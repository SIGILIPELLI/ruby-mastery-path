# 02 · Databases with ActiveRecord

ActiveRecord is Rails' object-relational mapper, but it's a gem you can
`require` and use in any Ruby script — no Rails app needed. It maps
database tables to Ruby classes and rows to objects, so you query with
Ruby method calls instead of writing raw SQL for everyday work.

## Connecting and defining a schema

```ruby
require 'active_record'

ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: ':memory:'
)

ActiveRecord::Schema.define do
  create_table :authors do |t|
    t.string :name
  end

  create_table :books do |t|
    t.string  :title
    t.integer :author_id
    t.integer :year
  end
end
```

`:memory:` gives you a throwaway SQLite database that lives only for the
process — perfect for scripts, tests, and this lesson. A real app points
`database:` at a file or a Postgres/MySQL connection instead.

## Models and associations

A model is a plain class inheriting from `ActiveRecord::Base`. ActiveRecord
reads the table's columns automatically — you never declare attributes by
hand:

```ruby
class Author < ActiveRecord::Base
  has_many :books
end

class Book < ActiveRecord::Base
  belongs_to :author
  validates :title, presence: true
end
```

`has_many` / `belongs_to` set up the relationship in both directions,
matched by the `author_id` foreign key column on `books`.

## Creating and querying records

```ruby
a = Author.create!(name: "Ursula K. Le Guin")
a.books.create!(title: "The Left Hand of Darkness", year: 1969)
a.books.create!(title: "The Dispossessed", year: 1974)

puts Book.count
puts a.books.pluck(:title).inspect
puts Book.where("year < ?", 1970).first.title
```

Captured output:

```text
2
["The Left Hand of Darkness", "The Dispossessed"]
The Left Hand of Darkness
```

`create!` raises on failure (missing required data, failed validation);
plain `create` returns an unsaved object instead and leaves you to check
`.persisted?` or `.errors`. `pluck` fetches only the named column(s) as
a fast, low-memory array instead of loading full model objects.

## Validations

```ruby
b = Book.new
puts b.valid?
puts b.errors.full_messages.inspect
```

```text
false
["Title can't be blank"]
```

`valid?` runs all validations and populates `errors` without touching the
database. Validations only run automatically on `save`/`create` — calling
`update_attribute` on some Rails versions or writing to the database with
raw SQL bypasses them entirely.

## Migrations (the file-based version)

Outside a one-off script, schema changes usually live in versioned
migration files instead of an inline `Schema.define` block:

```ruby
class CreateBooks < ActiveRecord::Migration[7.1]
  def change
    create_table :books do |t|
      t.string  :title
      t.integer :year
      t.references :author
      t.timestamps
    end
  end
end
```

`t.references :author` is shorthand for an `author_id` integer column plus
an index. `t.timestamps` adds `created_at`/`updated_at`, which
ActiveRecord maintains for you automatically on every save.

## Eager loading — avoiding N+1 queries

```ruby
# N+1: one query for books, then one more query PER book for its author
Book.all.each { |book| puts book.author.name }

# One extra query total, joined in up front
Book.includes(:author).each { |book| puts book.author.name }
```

`includes` loads the association ahead of time in a second batched query
(or a JOIN) instead of firing a fresh query every time you touch
`.author` inside the loop — the difference between 1 query and N+1 queries
matters enormously once N is in the thousands.

## ActiveRecord-specific traps

- **`save` returns `false` on failure; it does not raise.** Code that
  assumes `save` always succeeds silently continues with an invalid
  record. Use `save!`/`create!` when a failure should stop execution, or
  always check the boolean return value.
- **Mass assignment takes whatever hash you hand it.** `Book.create(params)`
  with an unfiltered `params` hash from user input can set columns you
  never intended to expose. Always pass an explicit, whitelisted hash.
- **`destroy` vs `delete`**: `destroy` instantiates the record, runs
  callbacks and `dependent:` association cleanup, then deletes it.
  `delete` (and `delete_all`) skip all of that and issue raw SQL directly
  — faster, but silently orphans associated data if you relied on
  callbacks.
- **Query methods are lazy.** `Book.where(year: 1969)` doesn't hit the
  database until you enumerate it (`each`, `to_a`, `first`, ...).
  Assigning it to a variable and reusing it does NOT cache a stale
  snapshot automatically re-running the query each time you evaluate it —
  but it also means a `where` chained onto it later still applies before
  execution, which surprises people expecting eager evaluation.
- **`:memory:` SQLite databases are per-connection.** If your script opens
  more than one connection, each gets its own empty in-memory database —
  data written on one is invisible to the other. Use a real file (even a
  temp one) if you need multiple connections to see the same data.

## Cheat sheet

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

## Exercise

Using the `Author`/`Book` schema above, in a script:

1. Create three authors, each with two books, varying `year`.
2. Write a query that returns all books published before 1980, ordered
   newest-first, printing `"title (year) — author name"` for each using
   `includes(:author)` to avoid N+1 queries.
3. Add a `validates :year, numericality: { greater_than: 0 }` to `Book`
   and demonstrate a `Book.new(title: "X", year: -5).valid?` returning
   `false` with the expected error message.
4. Print the total count of books per author using
   `Author.joins(:books).group(:name).count`.

Run the script and paste the printed output.
