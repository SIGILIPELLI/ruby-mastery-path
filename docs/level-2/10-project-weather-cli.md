# 10 · Project — Weather CLI

A capstone project pulling together everything from Level 2: OOP with
modules and `Comparable`, blocks/`Enumerable`, custom exception classes,
and RSpec tests with doubles. You'll build a command-line tool that looks
up the current weather for one or more cities using **Open-Meteo**, a
free weather API that requires **no API key or signup**.

## What you'll build

A `bin/weather` script that:

- Accepts one or more city names as command-line arguments
- Geocodes each city name to a latitude/longitude
- Fetches current weather for each location
- Skips (rather than crashes on) a city that fails, and reports why
- Prints all results sorted warmest-to-coldest, plus the average temperature
- Ships with an RSpec suite that runs with **no network access required**

## Project layout

```text
weather_cli/
    Gemfile
    bin/
        weather
    lib/
        weather_cli.rb
        weather_cli/
            errors.rb
            geocoder.rb
            weather_client.rb
            weather_codes.rb
            forecast.rb
    spec/
        spec_helper.rb
        geocoder_spec.rb
        weather_client_spec.rb
        forecast_spec.rb
        weather_codes_spec.rb
```

## Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

gem "rspec", "~> 3.12", group: :test
```

## lib/weather_cli/errors.rb — a small exception hierarchy

```ruby
module WeatherCli
  class Error < StandardError; end
  class GeocodingError < Error; end
  class ApiError < Error; end
end
```

Both failure modes (bad city name, bad weather response) become subclasses
of one `WeatherCli::Error`, so `bin/weather` can rescue broadly while tests
and internals can rescue narrowly. See
[Exception Handling](03-exception-handling.md) for why this beats raising
plain `RuntimeError` everywhere.

## lib/weather_cli/geocoder.rb — city name to coordinates

```ruby
require "net/http"
require "uri"
require "json"
require_relative "errors"

module WeatherCli
  # Turns a city name into latitude/longitude using Open-Meteo's free,
  # no-API-key geocoding endpoint.
  class Geocoder
    GEOCODE_URL = "https://geocoding-api.open-meteo.com/v1/search"

    Location = Struct.new(:name, :country, :latitude, :longitude)

    def find(city_name)
      uri = URI(GEOCODE_URL)
      uri.query = URI.encode_www_form(name: city_name, count: 1)

      response = Net::HTTP.get_response(uri)
      raise GeocodingError, "Geocoding request failed: #{response.code}" unless response.is_a?(Net::HTTPSuccess)

      data = JSON.parse(response.body)
      result = data.dig("results", 0)
      raise GeocodingError, "No location found for #{city_name.inspect}" unless result

      Location.new(result["name"], result["country"], result["latitude"], result["longitude"])
    rescue JSON::ParserError => e
      raise GeocodingError, "Invalid geocoding response: #{e.message}"
    end
  end
end
```

## lib/weather_cli/weather_client.rb — coordinates to current weather

```ruby
require "net/http"
require "uri"
require "json"
require_relative "errors"

module WeatherCli
  # Fetches current weather for a latitude/longitude from Open-Meteo's free,
  # no-API-key forecast endpoint.
  class WeatherClient
    FORECAST_URL = "https://api.open-meteo.com/v1/forecast"

    CurrentWeather = Struct.new(:temperature, :windspeed, :weathercode, :time)

    def initialize(open_timeout: 5, read_timeout: 5)
      @open_timeout = open_timeout
      @read_timeout = read_timeout
    end

    def current_weather(latitude, longitude)
      uri = URI(FORECAST_URL)
      uri.query = URI.encode_www_form(
        latitude: latitude,
        longitude: longitude,
        current_weather: true
      )

      response = get(uri)
      raise ApiError, "Weather request failed: #{response.code}" unless response.is_a?(Net::HTTPSuccess)

      data = JSON.parse(response.body).fetch("current_weather")
      CurrentWeather.new(data["temperature"], data["windspeed"], data["weathercode"], data["time"])
    rescue JSON::ParserError => e
      raise ApiError, "Invalid weather response: #{e.message}"
    rescue KeyError => e
      raise ApiError, "Unexpected weather response shape: #{e.message}"
    end

    private

    def get(uri)
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = uri.scheme == "https"
      http.open_timeout = @open_timeout
      http.read_timeout = @read_timeout
      http.get(uri.request_uri)
    rescue Net::OpenTimeout, Net::ReadTimeout => e
      raise ApiError, "Weather request timed out: #{e.class}"
    end
  end
end
```

## lib/weather_cli/weather_codes.rb — decoding WMO weather codes

```ruby
module WeatherCli
  # Open-Meteo reports weather as a numeric WMO code. This module translates
  # a code into a short human-readable description, using a Hash with a
  # default block so unrecognized codes never raise -- they just describe
  # themselves as unknown instead of crashing the CLI.
  module WeatherCodes
    DESCRIPTIONS = Hash.new { |_hash, code| "Unknown conditions (code #{code})" }.merge(
      0 => "Clear sky",
      1 => "Mainly clear",
      2 => "Partly cloudy",
      3 => "Overcast",
      45 => "Fog",
      51 => "Light drizzle",
      61 => "Slight rain",
      63 => "Moderate rain",
      65 => "Heavy rain",
      71 => "Slight snow",
      80 => "Rain showers",
      95 => "Thunderstorm"
    )

    def self.describe(code)
      DESCRIPTIONS[code]
    end
  end
end
```

## lib/weather_cli/forecast.rb — OOP + Comparable

```ruby
require_relative "weather_codes"

module WeatherCli
  # Combines a location with its current weather reading. Comparable is
  # mixed in so a list of Forecasts can be sorted/min/max'd by temperature
  # with no extra code -- see Enumerable & Comparable for how this works.
  class Forecast
    include Comparable

    attr_reader :city, :country, :temperature, :windspeed, :description

    def initialize(city:, country:, temperature:, windspeed:, weathercode:)
      @city = city
      @country = country
      @temperature = temperature
      @windspeed = windspeed
      @description = WeatherCodes.describe(weathercode)
    end

    def <=>(other)
      temperature <=> other.temperature
    end

    def to_s
      format("%-15s %-15s %5.1f°C  %-18s wind %.0f km/h", city, country, temperature, description, windspeed)
    end
  end
end
```

## lib/weather_cli.rb — tying it together

```ruby
require_relative "weather_cli/errors"
require_relative "weather_cli/geocoder"
require_relative "weather_cli/weather_client"
require_relative "weather_cli/weather_codes"
require_relative "weather_cli/forecast"

module WeatherCli
  # Ties Geocoder + WeatherClient together to build a Forecast for a city
  # name, and rescues either failing into one common WeatherCli::Error so
  # callers only need to handle one exception type.
  class Report
    def initialize(geocoder: Geocoder.new, weather_client: WeatherClient.new)
      @geocoder = geocoder
      @weather_client = weather_client
    end

    def forecast_for(city_name)
      location = @geocoder.find(city_name)
      current = @weather_client.current_weather(location.latitude, location.longitude)

      Forecast.new(
        city: location.name,
        country: location.country,
        temperature: current.temperature,
        windspeed: current.windspeed,
        weathercode: current.weathercode
      )
    end

    # Builds a Forecast for every city name given. A city that fails (typo,
    # network hiccup) doesn't abort the whole run -- its error is yielded
    # to the caller instead of raised, via the optional block.
    def forecast_for_all(city_names)
      city_names.map do |city_name|
        begin
          forecast_for(city_name)
        rescue Error => e
          yield(city_name, e) if block_given?
          nil
        end
      end.compact
    end
  end
end
```

## bin/weather — the CLI entry point

```ruby
#!/usr/bin/env ruby
require_relative "../lib/weather_cli"

if ARGV.empty?
  puts "Usage: bin/weather <city> [city2] [city3] ..."
  exit 1
end

report = WeatherCli::Report.new

forecasts = report.forecast_for_all(ARGV) do |city_name, error|
  puts "Skipping #{city_name.inspect}: #{error.message}"
end

if forecasts.empty?
  puts "No forecasts could be retrieved."
  exit 1
end

puts "\nCurrent weather:"
forecasts.sort.reverse_each { |forecast| puts forecast }

warmest = forecasts.max
coldest = forecasts.min
average = forecasts.sum(&:temperature) / forecasts.size.to_f

puts "\nWarmest: #{warmest.city} (#{warmest.temperature}°C)"
puts "Coldest: #{coldest.city} (#{coldest.temperature}°C)"
puts format("Average: %.1f°C across %d cities", average, forecasts.size)
```

Note `forecasts.sort` and `.max`/`.min` all work automatically because
`Forecast` includes `Comparable` — no custom sorting logic needed here at
all (see [Enumerable & Comparable](09-enumerable-comparable.md)).

## Running it

```bash
chmod +x bin/weather
ruby bin/weather London Tokyo Paris "Not A Real City XYZ123"
```

```text
Skipping "Not A Real City XYZ123": No location found for "Not A Real City XYZ123"

Current weather:
Tokyo           Japan            30.7°C  Mainly clear       wind 5 km/h
Paris           France           19.6°C  Clear sky          wind 7 km/h
London          United Kingdom   18.9°C  Partly cloudy      wind 14 km/h

Warmest: Tokyo (30.7°C)
Coldest: London (18.9°C)
Average: 23.1°C across 3 cities
```

(Exact temperatures will differ when you run it — this hits a live
weather API. No API key or `.env` file is needed anywhere in this
project.)

## Testing without hitting the network

The whole point of wrapping HTTP calls in `Geocoder`/`WeatherClient` is
that the RSpec suite can fake the HTTP layer and never make a real
request — tests stay fast and deterministic:

```ruby
# spec/spec_helper.rb
$LOAD_PATH.unshift(File.expand_path("../lib", __dir__))
require "weather_cli"

# A tiny fake HTTP response shared by the Geocoder and WeatherClient specs,
# so neither one ever makes a real network call. `is_a?` is overridden so
# `response.is_a?(Net::HTTPSuccess)` in the production code behaves exactly
# as it would against a real Net::HTTP response object.
FakeResponse = Struct.new(:code, :body) do
  def is_a?(klass)
    klass == Net::HTTPSuccess ? code == "200" : super
  end
end
```

```ruby
# spec/geocoder_spec.rb
require "spec_helper"

RSpec.describe WeatherCli::Geocoder do
  subject(:geocoder) { described_class.new }

  it "returns a Location built from the first search result" do
    body = { results: [{ name: "Paris", country: "France", latitude: 48.85, longitude: 2.35 }] }.to_json
    allow(Net::HTTP).to receive(:get_response).and_return(FakeResponse.new("200", body))

    location = geocoder.find("Paris")

    expect(location.name).to eq("Paris")
    expect(location.country).to eq("France")
    expect(location.latitude).to eq(48.85)
  end

  it "raises GeocodingError when no results are found" do
    body = { results: [] }.to_json
    allow(Net::HTTP).to receive(:get_response).and_return(FakeResponse.new("200", body))

    expect { geocoder.find("Nowhereville") }.to raise_error(WeatherCli::GeocodingError, /No location found/)
  end

  it "raises GeocodingError on a non-success HTTP status" do
    allow(Net::HTTP).to receive(:get_response).and_return(FakeResponse.new("500", ""))

    expect { geocoder.find("Paris") }.to raise_error(WeatherCli::GeocodingError, /failed/)
  end

  it "raises GeocodingError on malformed JSON" do
    allow(Net::HTTP).to receive(:get_response).and_return(FakeResponse.new("200", "not json"))

    expect { geocoder.find("Paris") }.to raise_error(WeatherCli::GeocodingError, /Invalid geocoding response/)
  end
end
```

```ruby
# spec/weather_client_spec.rb
require "spec_helper"

RSpec.describe WeatherCli::WeatherClient do
  subject(:client) { described_class.new }

  # A double standing in for the Net::HTTP connection object, so no real
  # socket is ever opened during the test run.
  let(:fake_http) { instance_double(Net::HTTP) }

  before do
    allow(Net::HTTP).to receive(:new).and_return(fake_http)
    allow(fake_http).to receive(:use_ssl=)
    allow(fake_http).to receive(:open_timeout=)
    allow(fake_http).to receive(:read_timeout=)
  end

  it "parses a successful response into a CurrentWeather struct" do
    body = { current_weather: { temperature: 21.4, windspeed: 12.0, weathercode: 2, time: "2024-01-01T00:00" } }.to_json
    allow(fake_http).to receive(:get).and_return(FakeResponse.new("200", body))

    weather = client.current_weather(48.85, 2.35)

    expect(weather.temperature).to eq(21.4)
    expect(weather.weathercode).to eq(2)
  end

  it "raises ApiError on a non-success response" do
    allow(fake_http).to receive(:get).and_return(FakeResponse.new("503", ""))

    expect { client.current_weather(48.85, 2.35) }.to raise_error(WeatherCli::ApiError, /failed/)
  end

  it "raises ApiError when the response body is not valid JSON" do
    allow(fake_http).to receive(:get).and_return(FakeResponse.new("200", "not json"))

    expect { client.current_weather(48.85, 2.35) }.to raise_error(WeatherCli::ApiError, /Invalid weather response/)
  end

  it "raises ApiError on a timeout" do
    allow(fake_http).to receive(:get).and_raise(Net::ReadTimeout)

    expect { client.current_weather(48.85, 2.35) }.to raise_error(WeatherCli::ApiError, /timed out/)
  end
end
```

```ruby
# spec/forecast_spec.rb
require "spec_helper"

RSpec.describe WeatherCli::Forecast do
  def build(city, temperature)
    described_class.new(city: city, country: "Testland", temperature: temperature, windspeed: 10, weathercode: 0)
  end

  it "exposes a human-readable weather description" do
    forecast = build("Testville", 20.0)
    expect(forecast.description).to eq("Clear sky")
  end

  it "compares forecasts by temperature via Comparable" do
    cold = build("Reykjavik", 4.0)
    hot = build("Cairo", 38.0)

    expect(cold).to be < hot
    expect([hot, cold].sort).to eq([cold, hot])
    expect([hot, cold].max).to eq(hot)
  end
end
```

```ruby
# spec/weather_codes_spec.rb
require "spec_helper"

RSpec.describe WeatherCli::WeatherCodes do
  it "describes a known code" do
    expect(described_class.describe(0)).to eq("Clear sky")
  end

  it "falls back to a generic description for an unknown code" do
    expect(described_class.describe(999)).to eq("Unknown conditions (code 999)")
  end
end
```

Run the whole suite with:

```bash
bundle exec rspec
```

```text
............

Finished in 0.01 seconds (files took 0.09 seconds to load)
12 examples, 0 failures
```

All 12 examples pass without ever touching the network — exactly the
behavior you want from a test suite: fast, deterministic, and runnable
offline or in CI.

## Stretch goals

- Add a `--units imperial` flag that converts Celsius to Fahrenheit before
  printing (Open-Meteo also accepts a `temperature_unit=fahrenheit` query
  parameter directly, if you'd rather push the conversion server-side).
- Cache geocoding results to a local JSON file with `File` (see
  [File I/O](04-file-io.md)) so repeated lookups for the same city skip the
  network round-trip.
- Package this as an installable gem (see
  [Gem Development Basics](08-gem-development.md)) so `bin/weather` becomes
  a global `weather` command after `gem install`.
- Add a `forecast_days` option using Open-Meteo's `daily` parameters, and
  print a 3-day outlook per city instead of just the current reading.

Completing this project means you're ready for **Level 3 · Advanced**.
