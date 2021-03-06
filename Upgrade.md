See the [Changelog](changelog) for a complete list of changes from VCR
1.x to 2.0. This file simply lists the most pertinent ones to upgrading.

## Supported Rubies

Ruby 1.8.6 and 1.9.1 are no longer supported.

## Configuration Changes

In VCR 1.x, your configuration block would be something like this:

``` ruby
VCR.config do |c|
  c.cassette_library_dir = 'cassettes'
  c.stub_with :fakeweb, :typhoeus
end
```

This will continue to work in VCR 2.0 but will generate deprecation
warnings. Instead, you should change this to:

``` ruby
VCR.configure do |c|
  c.cassette_library_dir = 'cassettes'
  c.hook_into :fakeweb, :typhoeus
end
```

## New Cassette Format

The cassette format has changed between VCR 1.x and VCR 2.0.
VCR 1.x cassettes cannot be used with VCR 2.0.

The easiest way to upgrade is to simply delete your cassettes and
re-record all of them. VCR also provides a rake task that attempts
to upgrade your 1.x cassettes to the new 2.0 format. To use it, add
the following line to your Rakefile:

``` ruby
load 'vcr/tasks/vcr.rake'
```

Then run `rake vcr:migrate_cassettes DIR=path/to/your/cassettes/directory` to
upgrade your cassettes. Note that this rake task may be unable to
upgrade some cassettes that make extensive use of ERB. In addition, now
that VCR 2.0 does less normalization then before, it may not be able to
migrate the cassette perfectly. It's recommended that you delete and
re-record your cassettes if you are able.

## Custom Request Matchers

VCR 2.0 allows any callable (an object that responds to #call, such as a lambda)
to be used as a request matcher:

``` ruby
VCR.configure do |c|
  c.register_request_matcher :port do |request_1, request_2|
    URI(request_1.uri).port == URI(request_2.uri).port
  end
end
```

In addition, a helper method is provided for generating a custom
matcher that ignores one or more query parameters:

``` ruby
uri_without_timestamp = VCR.request_matchers.uri_without_param(:timestamp)
VCR.configure do |c|
  c.register_request_matcher(:uri_without_timestamp, &uri_without_timestamp)
end
```

## Custom Serializers

VCR 2.0 supports multiple serializers. `:yaml`, `:json`, `:psych` and
`:syck` are supported out of the box, and it's easy to implement your
own:

``` ruby
VCR.use_cassette("example", :serialize_with => :json) do
  # make an HTTP request
end

marshal_serializer = Object.new
marshal_serializer.instance_eval do
  def file_extension
    "marsh"
  end

  def serialize(hash)
    Marshal.dump(hash)
  end

  def deserialize(string)
    Marshal.load(string)
  end
end

VCR.configure do |c|
  c.cassette_serializers[:marshal] = serializer
  c.default_cassette_options = { :serialize_with => :marshal }
end
```

## Request Hooks

VCR 2.0 has new request hooks, allowing you to inject custom logic
before an HTTP request, after an HTTP request, or around an HTTP
request:

``` ruby
VCR.configure do |c|
  c.before_http_request do |request|
    # do something with the request
  end

  c.after_http_request do |request, response|
    # do something with the request or response
  end

  # around_http_request only works on ruby 1.9
  VCR.configure do |c|
    c.around_http_request do |request|
      uri = URI(request.uri)
      if uri.host == 'api.geocoder.com'
        # extract an address like "1700 E Pine St, Seattle, WA"
        # from a query like "address=1700+E+Pine+St%2C+Seattle%2C+WA"
        address = CGI.unescape(uri.query.split('=').last)
        VCR.use_cassette("geocoding/#{address}", &request)
      else
        request.proceed
      end
    end
  end
end
```

## Ignore a Request Based on Anything

You can now define what requests get ignored using a block. This
gives you the flexibility to ignore a requets based on anything.

``` ruby
VCR.configure do |c|
  c.ignore_request do |request|
    uri = URI(request.uri)
    uri.host == 'localhost' && uri.port == 7500
  end
end
```

## Integration with RSpec 2 Metadata

VCR can integrate directly with RSpec metadata:

``` ruby
VCR.configure do |c|
  c.configure_rspec_metadata!
end

RSpec.configure do |c|
  # so we can use `:vcr` rather than `:vcr => true`;
  # in RSpec 3 this will no longer be necessary.
  c.treat_symbols_as_metadata_keys_with_true_values = true
end

# apply it to an example group
describe MyAPIWrapper, :vcr do
end

describe MyAPIWrapper do
  # apply it to an individual example
  it "does something", :vcr do
  end

  # set some cassette options
  it "does something", :vcr => { :record => :new_episodes } do
  end

  # override the cassette name
  it "does something", :vcr => { :cassette_name => "something" } do
  end
end
```

## Improved Faraday Integration

VCR 1.x integrated with Faraday but required that you insert
`VCR::Middleware::Faraday` into your middleware stack and configure
`stub_with :faraday`. VCR 2 now takes care of inserting itself
into the Faraday middleware stack if you configure `hook_into :faraday`.

## Improved Unhandled Error Messages

When VCR is unsure how to handle a request, the error message now contains
suggestions for how you can configure VCR or your test so it can handle
the request.

## Debug Logger

VCR 2.0 has a new configuration option that will turn on a logging mode
so you can get more insight into what VCR is doing, for troubleshooting
purposes:

``` ruby
VCR.configure do |c|
  c.debug_logger = File.open('log/vcr.log')
  # or...
  c.debug_logger = $stderr
end
```

## Playback Changes

In VCR 1.x, a single HTTP interaction could be played back multiple
times. This was mostly due to how VCR was implemented using FakeWeb
and WebMock, and was not really by design. It's more in keeping with
the philosophy of VCR to record the entire sequence of HTTP interactions
(including the duplicate requests). In VCR 2, each recorded HTTP
interaction can only be played back once unless you use the new
`:allow_playback_repeats` option.

