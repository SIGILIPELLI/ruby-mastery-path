# 01 · Building Web Apps with Sinatra

Every Ruby web framework you've heard of — including Rails — is built on
**Rack**, a simple interface between web servers and Ruby applications.
Sinatra is the thinnest possible layer on top of Rack: define a route,
write a block, return a string. No controllers, no folders, no
convention-over-configuration magic. That makes it the fastest way to see
how HTTP actually maps onto Ruby code.

## Your first Sinatra app

```ruby
# app.rb
require 'sinatra'

get '/' do
  "Hello, Sinatra!"
end
```

Running `ruby app.rb` starts a server on port 4567 by default (WEBrick or
Puma, whichever is installed). Visiting `/` in a browser runs the block
and sends its return value back as the response body — in Sinatra, **the
last expression in a route block is the response**, exactly like a method.

## Routes and dynamic segments

Routes are declared with an HTTP verb method (`get`, `post`, `put`,
`patch`, `delete`) and a path. A path segment starting with `:` becomes a
named parameter, available through the `params` hash:

```ruby
get '/greet/:name' do
  "Hello, #{params[:name]}!"
end
```

A request to `/greet/Ruby` runs the block with `params[:name] == "Ruby"`
and returns `"Hello, Ruby!"`.

## Reading form and query data

`params` merges route segments, query-string parameters, and submitted
form fields into a single hash — you don't need to distinguish where a
value came from to read it:

```ruby
post '/echo' do
  "You sent: #{params[:message]}"
end
```

A `POST /echo` with body `message=hi+there` returns `"You sent: hi
there"`. A `GET /echo?message=hi` would land in the same `params[:message]`
if you also defined a `get '/echo'` route.

## Testing routes without a real server

You rarely want to boot an actual HTTP server to check a route. `rack-test`
drives your app in-process by sending fake Rack requests directly to it,
which is both faster and deterministic:

```ruby
# test_app.rb
require 'rack/test'
require_relative 'app'

class Tester
  include Rack::Test::Methods
  def app; Sinatra::Application; end
end

t = Tester.new

t.get '/'
puts t.last_response.status  # 200
puts t.last_response.body    # Hello, Sinatra!

t.get '/greet/Ruby'
puts t.last_response.body    # Hello, Ruby!

t.post '/echo', message: 'hi there'
puts t.last_response.body    # You sent: hi there
```

Captured output from running `ruby test_app.rb`:

```text
200
Hello, Sinatra!
Hello, Ruby!
You sent: hi there
```

## Views with ERB

Returning raw strings works for a demo, but real apps render templates.
Sinatra looks for view files in a `views/` directory next to your app
file and renders them with `erb`:

```ruby
# app.rb
get '/profile/:name' do
  erb :profile, locals: { name: params[:name] }
end
```

```erb
<!-- views/profile.erb -->
<h1>Profile: <%= name %></h1>
<p>Member since <%= Time.now.year %></p>
```

`<%= %>` interpolates a Ruby expression into the output; plain `<% %>`
runs Ruby without printing anything (useful for `if`/`each` blocks).

## Before filters and halting

A `before` block runs ahead of every matching route — the classic place
to check authentication or set shared state:

```ruby
before '/admin/*' do
  halt 401, "Not authorized" unless params[:token] == "secret"
end

get '/admin/dashboard' do
  "Welcome, admin"
end
```

`halt` immediately stops processing and sends the given status and body —
the route block below never runs if the `before` filter halts first.

## Sinatra-specific traps

- **Route order matters.** Sinatra matches routes top-to-bottom and stops
  at the first match. A generic `get '/:id'` declared before
  `get '/new'` will swallow requests meant for `/new` (`params[:id]` would
  be `"new"`). Put specific routes first.
- **`params` keys can be strings *or* symbols depending on source.**
  Route-segment params are always symbols, but Sinatra also indifferently
  duplicates them as strings in some setups. Don't assume; use
  `params[:name]` consistently rather than mixing `params["name"]`.
- **The classic single-file style (`require 'sinatra'`) is process-global**
  — there's only one `Sinatra::Application`. For multiple independent apps
  in one process, use the modular style (`class App < Sinatra::Base`)
  instead, which is also what `rack-test` needs to target more than one
  app.
- **`halt` inside a block silently ends the block** — code after a
  triggered `halt` never executes. It's easy to forget this isn't just
  `return`; it also skips any `after` filters' assumptions about
  reaching a normal response.
- **Sinatra 4.x enables `host_authorization` by default**, which rejects
  requests whose `Host` header isn't recognized — including requests from
  `rack-test` unless you set `set :host_authorization, {
  permitted_hosts: [] }` (fine for tests/dev, tighten it for real
  deployments).

## Cheat sheet

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

## Exercise

Build a tiny Sinatra "notes" app in a single `app.rb`:

1. An in-memory `NOTES = []` array of hashes (`{ id:, text: }`).
2. `GET /notes` returns all notes joined by newlines (empty array →
   `"No notes yet"`).
3. `POST /notes` reads `params[:text]`, appends a new note with an
   auto-incrementing `id`, and returns `"Added note #\#{id}"`.
4. `GET /notes/:id` returns the matching note's text, or halts with
   `404, "Not found"` if no note has that id.
5. Write a `rack-test` script that posts two notes, then fetches both
   `/notes` and `/notes/:id` for each, printing the responses.

Run your test script and confirm the printed output matches what you'd
expect from the routes you wrote.
