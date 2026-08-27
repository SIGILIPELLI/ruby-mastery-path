# 03 · Security Best Practices

Most Ruby security bugs come from trusting input that shouldn't be
trusted: building a SQL query from a string, assigning whatever
attributes a request sent, or checking a secret into version control.
None of these require exotic knowledge to avoid — they require a
consistent habit of treating anything from outside your process as
hostile until proven otherwise.

## SQL injection — string interpolation vs. parameterization

```ruby
User.create!(name: "alice", role: "user")
User.create!(name: "bob' OR '1'='1", role: "admin")

# UNSAFE: string interpolation into SQL
name_input = "alice' OR '1'='1"
unsafe = User.where("name = '#{name_input}'")
puts "Unsafe query matched #{unsafe.count} rows"

# SAFE: parameterized query
safe = User.where("name = ?", name_input)
puts "Safe query matched #{safe.count} rows"
```

Captured output:

```text
Unsafe query matched 2 rows
Safe query matched 0 rows
```

The interpolated version builds the literal SQL string `name = 'alice'
OR '1'='1'` — the attacker-supplied `OR '1'='1'` turns the `WHERE` clause
into "match everything," returning both rows regardless of the intended
filter. The parameterized version (`"name = ?", name_input`) passes the
value through the database driver's own escaping, so the whole string —
quotes and all — is treated as *data*, matching (correctly) zero rows,
since no user is actually named `alice' OR '1'='1`.

**Rule: never build a query string via interpolation with untrusted
input.** Use `?` placeholders, named placeholders (`"name = :name",
name: input`), or ActiveRecord's hash form (`where(name: input)`, which
is always parameterized) instead.

## Mass assignment — filtering what's allowed to be set

```ruby
class SignupParams
  ALLOWED = [:name].freeze

  def self.permit(raw)
    raw.slice(*ALLOWED)
  end
end

dangerous_input = { name: "carol", role: "admin" }
safe_attrs = SignupParams.permit(dangerous_input)
u = User.create!(safe_attrs.merge(role: "user"))
puts "Created user with role: #{u.role}"
```

```text
Created user with role: user
```

Without the explicit `permit`/`slice` step, `User.create!(dangerous_input)`
would happily set `role: "admin"` from a value that arrived in an
attacker-controlled request body — a classic privilege-escalation bug
where a signup form silently accepts an unexpected `role` field it was
never meant to. Rails' `strong_parameters` (`params.require(:user).permit(:name)`)
is this exact pattern, built into the framework; a hand-rolled Sinatra
app needs to whitelist explicitly, as shown here.

## Secrets — never in source code

```ruby
# WRONG — the secret lives forever in git history, even after "removing" it
API_KEY = "sk_live_abc123..."

# RIGHT — read from the environment, provided outside the codebase
API_KEY = ENV.fetch("STRIPE_API_KEY")
```

`ENV.fetch` (as opposed to `ENV[...]`, which silently returns `nil` for a
missing key) raises immediately if the variable isn't set — failing loud
at boot is much better than a payment integration silently sending
`nil` as an API key and failing mysteriously later. Real secrets belong
in environment variables, a secrets manager, or an encrypted credentials
file (`config/credentials.yml.enc` in Rails) — never in a committed
file, and never hardcoded even "temporarily," because git history keeps
it forever unless you rewrite history and rotate the leaked secret.

## Password storage — always hash, never store plaintext

```ruby
require 'bcrypt'

password = "correct horse battery staple"
hashed = BCrypt::Password.create(password)
puts hashed.start_with?("$2a$")

stored = BCrypt::Password.new(hashed)
puts stored == "correct horse battery staple"
puts stored == "wrong guess"
```

`BCrypt::Password.create` produces a salted, one-way hash — the salt is
embedded in the output string itself, so you don't manage it separately.
`stored == candidate` re-hashes the candidate with the same salt and
compares, never decrypting anything (bcrypt hashes are not reversible by
design). Storing plaintext or even naively-SHA256'd passwords means a
single database leak exposes every user's real password; bcrypt's
deliberate slowness also makes brute-forcing the hash impractical at
scale.

## Cross-site scripting (XSS) — escape output, don't trust input

Any user-supplied string rendered into HTML must be escaped, or a value
like `<script>steal_cookies()</script>` submitted as, say, a display name
executes in every other user's browser who views that page:

```ruby
require 'erb'
name = "<script>alert('xss')</script>"
puts ERB::Util.html_escape(name)
# &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
```

Template engines (ERB's `<%= %>`, Rails views) escape by default — the
danger is specifically calling something like `raw(user_input)` or
`html_safe` on untrusted content to "fix" a rendering issue, which
disables the automatic escaping that was protecting you.

## Security-specific traps

- **`where("name = '#{x}'")` looks identical to the safe version at a
  glance** — the vulnerability is invisible in a quick code review
  unless you specifically look for string interpolation inside a raw SQL
  fragment.
- **`ENV[...]` returning `nil` for a typo'd variable name** fails
  silently rather than loudly — `ENV.fetch` with no default converts
  that into an immediate, obvious startup crash instead of a mysterious
  runtime failure hours later.
- **Comparing hashed passwords with `==` on raw strings** instead of
  going through `BCrypt::Password#==` compares the *hash* against the
  *hash*, which will never match since bcrypt embeds a random salt —
  always compare via the `BCrypt::Password` wrapper, not manually.
- **Logging full request parameters** in production logs can leak
  passwords, tokens, or credit card numbers verbatim into log files
  that are often less carefully access-controlled than the database
  itself — filter sensitive parameter keys before logging.
- **Trusting `params[:role]` or any client-controlled field for
  authorization decisions.** Authorization checks belong in
  server-controlled state (a database column set by an admin action),
  never in something the request itself supplied.

## Cheat sheet

| Risk | Wrong | Right |
|---|---|---|
| SQL injection | `where("x = '#{v}'")` | `where("x = ?", v)` or `where(x: v)` |
| Mass assignment | `Model.create(raw_params)` | `Model.create(raw_params.slice(*ALLOWED))` |
| Hardcoded secret | `KEY = "sk_live_..."` | `KEY = ENV.fetch("KEY")` |
| Plaintext password | `password == stored_plain` | `BCrypt::Password.new(hash) == password` |
| Unescaped output | `raw(user_input)` | default ERB/Rails auto-escaping |
| Silent missing config | `ENV["KEY"]` | `ENV.fetch("KEY")` |

## Exercise

1. Write a small `SearchUsers` class with a deliberately vulnerable
   `unsafe_by_name(name)` method using string interpolation, then a
   `safe_by_name(name)` using a parameterized query — demonstrate the
   injection succeeding against the first and failing (correctly
   returning no/expected rows) against the second, using an input like
   `"' OR '1'='1"`.
2. Write a `PermittedParams` helper used by a fake `POST /signup`
   handler that strips any key not in an explicit allowlist, and show a
   submitted `{ name: "x", is_admin: true }` losing `is_admin` before it
   ever reaches `User.create`.
3. Using the `bcrypt` gem, write a tiny `login(email, password)` function
   against an in-memory `{ email => bcrypt_hash }` store, returning
   `true`/`false` correctly for a right and a wrong password, without
   ever storing or comparing a plaintext password directly.
