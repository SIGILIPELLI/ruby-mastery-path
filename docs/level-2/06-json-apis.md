# 06 · Working with JSON/APIs

Most modern services exchange data as JSON over HTTP. Ruby's standard
library ships everything you need to both produce and consume it: the
`json` library for parsing/generating JSON, and `net/http` for making HTTP
requests — no extra gems required for the basics.

## Parsing JSON

```ruby
require "json"

raw = '{"name": "Ada", "age": 30, "skills": ["Ruby", "Math"]}'

data = JSON.parse(raw)
puts data.inspect
# {"name"=>"Ada", "age"=>30, "skills"=>["Ruby", "Math"]}
puts data["name"]   # Ada -- keys are strings by default
```

## symbolize_names — keys as symbols instead of strings

```ruby
data = JSON.parse(raw, symbolize_names: true)
puts data[:name]   # Ada -- now keys are symbols
puts data[:skills].inspect   # ["Ruby", "Math"]
```

Symbol keys read more naturally in Ruby code (`data[:name]` vs
`data["name"]`), but only symbolize data from sources you trust — parsing
attacker-controlled JSON with `symbolize_names: true` can, in older Ruby
versions, create unbounded numbers of symbols, which are never
garbage-collected.

## Generating JSON

```ruby
require "json"

person = { name: "Grace", age: 34, skills: ["COBOL", "Compilers"] }

puts person.to_json
# {"name":"Grace","age":34,"skills":["COBOL","Compilers"]}

puts JSON.pretty_generate(person)
# {
#   "name": "Grace",
#   "age": 34,
#   "skills": [
#     "COBOL",
#     "Compilers"
#   ]
# }
```

`to_json` works on any standard Ruby object (Hash, Array, String, Integer,
nil, true/false) once `require "json"` has been loaded — it's added as a
method on `Object` by the library.

## Handling malformed JSON

```ruby
begin
  JSON.parse("{not valid json")
rescue JSON::ParserError => e
  puts "Bad JSON: #{e.message.split("\n").first}"
end
# Bad JSON: unexpected token at '{not valid json'
```

Always wrap `JSON.parse` in a `rescue JSON::ParserError` (see
[Exception Handling](03-exception-handling.md)) when the input comes from
outside your program — a network response, a file, user input — since
you can't guarantee it will always be well-formed.

## Making an HTTP GET request with Net::HTTP

```ruby
require "net/http"
require "uri"
require "json"

uri = URI("https://api.github.com/repos/ruby/ruby")
response = Net::HTTP.get_response(uri)

if response.is_a?(Net::HTTPSuccess)
  data = JSON.parse(response.body)
  puts "#{data['full_name']} has #{data['stargazers_count']} stars"
else
  puts "Request failed: #{response.code} #{response.message}"
end
# ruby/ruby has 21000+ stars (exact count changes over time)
```

`Net::HTTP.get_response` is the simplest way to issue a one-off GET
request. It returns a response object, not raw text — always check
`response.code` (or the `is_a?(Net::HTTPSuccess)` shortcut above) before
trusting `response.body`, since a 404 or 500 response still has a body,
just not the one you wanted.

## A small reusable HTTP client class

For anything beyond a single request, wrap the boilerplate in a class:

```ruby
require "net/http"
require "uri"
require "json"

class ApiClient
  class RequestError < StandardError; end

  def initialize(base_url)
    @base_url = base_url
  end

  def get(path)
    uri = URI("#{@base_url}#{path}")
    response = Net::HTTP.get_response(uri)

    raise RequestError, "#{response.code} #{response.message}" unless response.is_a?(Net::HTTPSuccess)

    JSON.parse(response.body)
  rescue JSON::ParserError => e
    raise RequestError, "Invalid JSON in response: #{e.message}"
  end
end

client = ApiClient.new("https://api.github.com")
repo = client.get("/repos/ruby/ruby")
puts repo["full_name"]   # ruby/ruby
```

Wrapping errors in your own `RequestError` (see
[Exception Handling](03-exception-handling.md) for building exception
hierarchies) means callers of `ApiClient` only need to rescue one exception
type, regardless of whether the underlying failure was a bad status code
or malformed JSON.

## Setting a timeout — never make a request that can hang forever

```ruby
require "net/http"

uri = URI("https://api.github.com/repos/ruby/ruby")
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true
http.open_timeout = 5   # seconds to establish the connection
http.read_timeout = 5    # seconds to wait for a response

begin
  response = http.get(uri.request_uri)
  puts response.code
rescue Net::OpenTimeout, Net::ReadTimeout => e
  puts "Request timed out: #{e.class}"
end
```

Without an explicit timeout, a hung server can block your program
indefinitely — always set one for requests that leave your process,
especially in anything long-running like a web server or background job.

## POST requests with a JSON body

```ruby
require "net/http"
require "uri"
require "json"

uri = URI("https://httpbin.org/post")
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true

request = Net::HTTP::Post.new(uri, "Content-Type" => "application/json")
request.body = { title: "New post", body: "Hello" }.to_json

response = http.request(request)
result = JSON.parse(response.body)
puts result["json"]["title"]   # New post
```

## How It Actually Works

`JSON.parse` is a recursive-descent parser: it walks the input string
character by character, and every time it encounters `{`, `[`, a quote, or
a digit, it recurses into a sub-parser for that JSON type, building nested
Ruby `Hash`/`Array`/`String`/`Numeric` objects as it goes — there's no
intermediate schema, so the resulting object graph mirrors the JSON's
nesting exactly. `JSON.generate` (what `to_json` calls) walks the opposite
direction, recursively calling `#to_json` on nested objects, which is why
defining a custom `to_json` method on your own class lets it serialize
however you like. Underneath `Net::HTTP` or `HTTParty`, a request opens a
TCP socket via the OS, and — like file I/O — the actual `read`/`write` on
that socket releases MRI's GIL, letting other threads run Ruby while your
thread blocks waiting for the network round trip; this is the mechanism
that makes concurrent HTTP requests via multiple threads genuinely useful
in Ruby despite the GIL.

## Cheat sheet

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

## Exercise

Write a class `WeatherClient` (a preview of the level project) with a
method `current_temperature(latitude, longitude)` that calls
`https://api.open-meteo.com/v1/forecast?latitude=..&longitude=..&current_weather=true`
(a free, no-API-key weather API), parses the JSON response, and returns
just the `temperature` value from `current_weather`. Handle both a
non-success HTTP status and a `JSON::ParserError` by raising a single
custom `WeatherClient::Error` — you'll build this out fully in
[Project: Weather CLI](10-project-weather-cli.md).
