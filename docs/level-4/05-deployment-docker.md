# 05 · Deployment (Docker)

!!! note "Verification note"
    Docker isn't available in the environment these lessons were
    verified in, so the Dockerfiles and commands below were reviewed
    manually rather than actually built and run. The Ruby, Sinatra, and
    shell syntax follows the standard, well-documented patterns from
    Docker's own Ruby image documentation — treat this module's Ruby
    application code (which runs fine outside Docker) as verified, and
    the container-specific instructions as reviewed-not-executed.

A Docker image packages your Ruby app together with its exact runtime —
the Ruby interpreter version, system libraries, and gems — into one
artifact that runs identically on your laptop, in CI, and in production.
"Works on my machine" stops being a category of bug once the "machine"
is a container image everyone runs the same way.

## A minimal Dockerfile for a Sinatra app

```dockerfile
# Dockerfile
FROM ruby:3.3-slim

WORKDIR /app

# Install gems first, in their own layer, so Docker only re-runs
# `bundle install` when Gemfile/Gemfile.lock actually change —
# not on every code edit.
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

COPY . .

EXPOSE 4567
CMD ["ruby", "app.rb", "-o", "0.0.0.0"]
```

```text
$ docker build -t task-api .
$ docker run -p 4567:4567 task-api
```

`-o 0.0.0.0` matters: Sinatra binds to `localhost` by default, which
inside a container means "only reachable from inside the container
itself" — `0.0.0.0` binds to all interfaces so the port mapped out to
the host (`-p 4567:4567`) actually reaches the process.

## Layer caching — why `COPY Gemfile* / RUN bundle install` comes first

Docker builds an image in layers, each cached independently. If
`COPY . .` (copying the whole app) came *before* `bundle install`, then
editing any file — even a comment in a view template — would invalidate
the cache for the following `bundle install` layer, forcing a full
gem reinstall on every single build. Copying only `Gemfile`/`Gemfile.lock`
first means that layer's cache only invalidates when dependencies
actually change, which is the whole reason for the seemingly odd
two-step `COPY` in the Dockerfile above.

## Multi-stage builds — smaller production images

Compiling native gem extensions (like `bcrypt` or `sqlite3` in earlier
modules) needs build tools (`gcc`, headers) that bloat the final image
and aren't needed at runtime. A multi-stage build compiles in one stage
and copies only the result into a lean final image:

```dockerfile
# Stage 1: build
FROM ruby:3.3-slim AS builder
WORKDIR /app
RUN apt-get update -qq && apt-get install -y build-essential libsqlite3-dev
COPY Gemfile Gemfile.lock ./
RUN bundle install --deployment --without development test

# Stage 2: runtime
FROM ruby:3.3-slim
WORKDIR /app
RUN apt-get update -qq && apt-get install -y libsqlite3-0 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/vendor/bundle /app/vendor/bundle
COPY . .
ENV BUNDLE_PATH=vendor/bundle
EXPOSE 4567
CMD ["ruby", "app.rb", "-o", "0.0.0.0"]
```

The final image never contains `build-essential` or the `-dev` headers —
only the compiled gem binaries needed to actually run, plus the
lightweight `libsqlite3-0` runtime library instead of the full
`libsqlite3-dev` development package.

## Environment variables and configuration

Following the security module's rule against hardcoded secrets, a
containerized app reads config from the environment, injected at
`docker run` time rather than baked into the image:

```text
$ docker run -p 4567:4567 \
    -e DATABASE_URL="postgres://user:pass@db-host/mydb" \
    -e RACK_ENV="production" \
    task-api
```

```ruby
DATABASE_URL = ENV.fetch("DATABASE_URL")
```

The same image runs in staging and production unchanged — only the
environment variables passed at `docker run` (or in your orchestrator's
config) differ, which is the whole point of building one image and
promoting it through environments rather than rebuilding per-environment.

## docker-compose — app plus its database, together

A real app usually needs a database alongside it. `docker-compose`
describes both as one unit:

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "4567:4567"
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/task_api
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```text
$ docker-compose up
```

`depends_on` starts `db` before `web`, but does **not** wait for
Postgres to actually finish initializing and accept connections — a
common gotcha covered below. `db` as the hostname in `DATABASE_URL`
works because Compose puts both services on the same private network,
where each service is reachable by its service name.

## Docker-specific traps

- **`depends_on` only orders container *start*, not service
  *readiness*.** Postgres's container can be "running" for several
  seconds before it's actually accepting connections — a Rails/Sinatra
  app that connects immediately on boot can crash-loop against a
  database that technically "started" but isn't ready yet. Production
  setups add an explicit healthcheck or a retry-with-backoff on the
  app's database connection instead of assuming `depends_on` means
  "ready."
- **Binding to `localhost` instead of `0.0.0.0` inside the container.**
  The app appears to work when you `docker exec` into the container and
  curl it locally, but is completely unreachable from outside — because
  `localhost` inside a container refers only to that container's own
  loopback interface.
- **Baking secrets into the image with `ENV` in the Dockerfile** instead
  of passing them at `docker run`/compose time. Anyone who can pull or
  inspect the image (`docker history`) can read a value baked in with
  `ENV SECRET_KEY=abc123` directly in the Dockerfile.
- **Not pinning the base image tag.** `FROM ruby:latest` means the exact
  Ruby version running in production silently shifts whenever `latest`
  is rebuilt upstream — pin a specific version (`ruby:3.3-slim`, or even
  more precisely `ruby:3.3.4-slim`) so a build today and a build next
  month use the identical base.
- **Forgetting a `.dockerignore`.** Without one, `COPY . .` copies
  `.git`, `log/`, `tmp/`, and local `.env` files into the image —
  bloating it and potentially leaking local secrets that happened to be
  in an untracked `.env` file into a shipped image.

## Cheat sheet

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

## Exercise

1. Write a `Dockerfile` for the Level 3 REST API capstone project
   (Sinatra + ActiveRecord + SQLite), including a `.dockerignore`
   excluding at least `.git`, `spec/`, and any local `*.sqlite3` file.
2. Convert it to a multi-stage build, moving any native-extension build
   dependencies (SQLite's dev headers) into a separate `builder` stage.
3. Write a `docker-compose.yml` running the API alongside a `redis`
   service (even if unused by the app yet), demonstrating `depends_on`
   and an environment variable pointing at the Redis service's hostname.
4. In a comment at the bottom of your `docker-compose.yml`, explain what
   change you'd make to guarantee the app doesn't try to connect to
   Redis before it's actually ready to accept connections.
