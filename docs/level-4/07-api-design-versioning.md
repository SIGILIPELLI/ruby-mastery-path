# 07 · API Design & Versioning

An API is a contract: once external clients (a mobile app, another
team's service, a third-party integration) depend on a response shape,
changing that shape breaks them silently, often long after you've
forgotten the endpoint exists. Good API design front-loads decisions
that make it possible to evolve the API without breaking existing
consumers — versioning is the main tool for that.

## Resource-oriented URLs

A REST API models data as **resources**, addressed by URL, manipulated
via HTTP verbs:

```text
GET    /users          list users
GET    /users/:id      fetch one user
POST   /users          create a user
PATCH  /users/:id      partially update a user
PUT    /users/:id      replace a user
DELETE /users/:id      delete a user
```

The verb carries the action; the URL only ever names the resource. A URL
like `POST /users/1/delete` mixes an action into the path and duplicates
what `DELETE /users/1` already expresses — a sign the API is drifting
from REST conventions into RPC-style "call this function" URLs.

## URL-path versioning

The simplest, most visible versioning strategy: put the version directly
in the path.

```ruby
class VersionedAPI < Sinatra::Base
  before { content_type :json }

  get "/api/v1/users/:id" do
    { id: params[:id].to_i, name: "Ada Lovelace" }.to_json
  end

  get "/api/v2/users/:id" do
    { id: params[:id].to_i, full_name: "Ada Lovelace", email: "ada@example.com" }.to_json
  end
end
```

```text
$ curl /api/v1/users/1
{"id":1,"name":"Ada Lovelace"}

$ curl /api/v2/users/1
{"id":1,"full_name":"Ada Lovelace","email":"ada@example.com"}
```

`v2` renamed `name` to `full_name` and added `email` — a **breaking
change** for any client parsing `name`. Because it lives at a different
URL, existing `v1` clients keep working, untouched, indefinitely (or
until you formally deprecate and remove `v1` on a published timeline).

## Header-based versioning

An alternative keeps one URL and negotiates the version via the
`Accept` header — more RESTful in spirit (a URL should identify a
resource, not a version of a resource), at the cost of being less
discoverable/testable by just pasting a URL into a browser:

```ruby
get '/api/users/:id' do
  version = request.env['HTTP_ACCEPT'].to_s[/version=(\d+)/, 1] || "1"
  if version == "2"
    { id: params[:id].to_i, full_name: "Ada Lovelace" }.to_json
  else
    { id: params[:id].to_i, name: "Ada Lovelace" }.to_json
  end
end
```

```text
$ curl -H "Accept: application/vnd.myapi.v2+json;version=2" /api/users/1
{"id":1,"full_name":"Ada Lovelace"}
```

Captured output from running all three requests via `rack-test`:

```text
{"id":1,"name":"Ada Lovelace"}
{"id":1,"full_name":"Ada Lovelace","email":"ada@example.com"}
{"id":1,"full_name":"Ada Lovelace"}
```

## What counts as a breaking change

- **Breaking**: removing a field, renaming a field, changing a field's
  type (string → number), changing status codes for existing scenarios,
  making an optional request parameter required.
- **Non-breaking (usually safe without a version bump)**: adding a new
  optional field to a response, adding a new endpoint, adding a new
  optional request parameter with a sensible default.

The rule of thumb: if an existing client's code, written against the
current contract, would start behaving incorrectly (crash, misinterpret
data, silently drop information) without any code changes on their end,
it's breaking.

## Consistent error responses

A well-designed API has one predictable error shape across every
endpoint, so clients write one error-handling path instead of one per
endpoint:

```ruby
error 404 do
  content_type :json
  { error: { code: "not_found", message: "Resource not found" } }.to_json
end

error 422 do
  content_type :json
  { error: { code: "validation_failed", message: env['sinatra.error']&.message } }.to_json
end
```

Every error response sharing the same `{ error: { code:, message: } }`
envelope means a client can write one generic error handler instead of
parsing a different shape per status code or per endpoint.

## API-design-specific traps

- **Versioning too late.** Shipping `v1` with no versioning scheme at
  all, then needing a breaking change, forces an awkward retrofit
  (redirect old clients, guess at a version from other signals) instead
  of a clean `v1` → `v2` path that was planned from day one.
- **Maintaining "just one more" version forever.** Every live API
  version is code you have to keep working and testing — publish a
  deprecation timeline for old versions (a `Sunset` header, a changelog
  entry, direct notice to known consumers) rather than letting versions
  accumulate silently forever.
- **Inconsistent pluralization/casing across endpoints.** Mixing
  `/api/v1/user` (singular) with `/api/v1/orders` (plural), or
  `snake_case` fields in one response and `camelCase` in another, forces
  every client integration to special-case your API's own
  inconsistency.
- **Returning `200 OK` for a request that actually failed.** A response
  body containing `{"success": false, "error": "..."}` with an HTTP
  `200` status defeats HTTP-layer tooling (load balancers, monitoring,
  generic HTTP client error handling) that correctly expects a non-2xx
  status to signal failure.
- **Silent behavior changes with no version bump at all** — the worst
  case: mutating a field's meaning or type in place on the *same*
  version, breaking every consumer with zero warning and no rollback
  path other than reverting the deploy.

## Cheat sheet

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

## Exercise

1. Design and implement `v1` and `v2` of a `GET /api/vN/products/:id`
   endpoint where `v2` renames `price` (a plain number) to
   `price_cents` (an integer) — demonstrating that this is exactly the
   kind of change that requires a version bump rather than an in-place
   field addition.
2. Add a consistent Sinatra `error 404`/`error 422` handler pair using
   the shared `{ error: { code:, message: } }` envelope, and write a
   request spec confirming both error responses share the same top-level
   shape.
3. Write a short `CHANGELOG.md`-style entry documenting the `v1` → `v2`
   breaking change from step 1, including a "sunset" date for `v1` and
   a one-line migration note for API consumers.
