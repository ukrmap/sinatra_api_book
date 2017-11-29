Chapter #4. Documentation and Testing
=====================================

If you are building API you also need provide documentation (specification) for it. And you create tests that are guarantee of reliability and also specification. In this chapter we will try to combine documentation and tests.

We will build minimalistic internally hardcoded users API that we will use in others services for authentication.

## <a name="planning-interface"></a>Planning Interface

Users API will have one route that will return JSON representation of specific user. User (can be other web service) of our API should provide correct authentication token in HTTP header of request. Authentication token is some sequence of characters that is known only to the user and web service. Generally if token is expired user should request new token, we will not build this functionality - tokens will not expire for simplicity.

We will use two types of users: "AdminUser" and "RegularUser" (actually only one "AdminUser" and one "RegularUser").

Example of the service request:

    $ curl -X GET "localhost:4567/api/v1/users/me.json" -H "Authorization: OAuth user_token"

    {"user":{"type":"RegularUser"}}

Where string `user_token` should be replaced with correct user token. Token - is value of "Authorization" header and prefixed with string "OAuth" (for [OAuth](https://ru.wikipedia.org/wiki/OAuth) or [OAuth 2.0](http://oauth.net/2/)).
If token is not correct service should response with 401 HTTP code (Unauthorized) and appropriate error message in JSON format. If token is not provided service should response with 403 HTTP code (Forbidden) and also appropriate error message in JSON format.

## <a name="create-service"></a>Create Service

Create please folder `users` somewhere in the system, this would be the root folder for new service. Create file `Gemfile` in it with next content:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'rspec'
gem 'rack-test'
gem 'rspec_api_documentation'
```

Navigate to service root folder in terminal and run:

    $ bundle install

Create please file `service.rb` in folder `users` with next content:

```ruby
require "sinatra/main"

get "/api/v1/users/me.json" do
  content_type :json

  case request.env['HTTP_AUTHORIZATION']
  when nil then [403, '{"message":"Access Forbidden"}']
  when "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
    then '{"user":{"type":"AdminUser"}}'
  when "OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"
    then '{"user":{"type":"RegularUser"}}'
  else [401, '{"message":"Invalid or expired token"}']
  end
end
```

We added one URL-route (with block). Block contains the value of two tokens (that obviously will not expire) directly in the code. It returns user with type "AdminUser" or "RegularUser" depending on token. We can run service with command `ruby service.rb` and test it (from another terminal window or tab):

    $ curl -X GET "localhost:4567/api/v1/users/me.json" \
      -H "Authorization: OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"

    {"user":{"type":"RegularUser"}}

We split `curl` request line on two lines by symbol "\" (backslash).

We can also check service response if provided incorrect token or no token. Use `-i` flag to see HTTP response with headers.

    $ curl -i -X GET "localhost:4567/api/v1/users/me.json" \
      -H "Authorization: OAuth wrong_token"

    HTTP/1.1 401 Unauthorized
    Content-Type: application/json
    Content-Length: 38
    X-Content-Type-Options: nosniff
    Connection: keep-alive
    Server: thin

    {"message":"Invalid or expired token"}

    $ curl -i -X GET "localhost:4567/api/v1/users/me.json"

    HTTP/1.1 403 Forbidden
    Content-Type: application/json
    Content-Length: 30
    X-Content-Type-Options: nosniff
    Connection: keep-alive
    Server: thin

    {"message":"Access Forbidden"}

## <a name="create-tests"></a>Create Tests

Create please folder `spec` in folder `users` and file `service_spec.rb` in it with next content:

```ruby
require "spec_helper"

describe "Users Service" do
  describe "GET /api/v1/users/me.json" do
    it "retrieves admin user JSON representation of provided token of admin user" do
      header "Authorization", "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
      get "/api/v1/users/me.json"
      expect(last_response).to be_ok
      expect(last_response.body).to eq '{"user":{"type":"AdminUser"}}'
    end

    it "retrieves regular user JSON representation of provided token of regular user" do
      header "Authorization", "OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"
      get "/api/v1/users/me.json"
      expect(last_response).to be_ok
      expect(last_response.body).to eq '{"user":{"type":"RegularUser"}}'
    end

    it "responds with 401 status and JSON error message if access token expired or incorrect" do
      header "Authorization", "OAuth 7564e5ab2d46d5af38e99e5490eea2c86b96f6a638d77fa0b124125ed26347eb"
      get "/api/v1/users/me.json"
      expect(last_response.status).to eq 401
      expect(last_response.body).to eq '{"message":"Invalid or expired token"}'
    end

    it "responds with 403 status and JSON error message if access token not provided" do
      get "/api/v1/users/me.json"
      expect(last_response.status).to eq 403
      expect(last_response.body).to eq '{"message":"Access Forbidden"}'
    end
  end
end
```

And file `spec_helper.rb` (also in folder `spec`)

```ruby
require_relative "../service"
require "rack/test"

RSpec.configure do |config|
  config.include Rack::Test::Methods

  def app
    Sinatra::Application
  end
end
```

Now we can run our tests:

    $ rspec
    ....

    Finished in 0.06302 seconds (files took 0.3102 seconds to load)
    4 examples, 0 failures

## <a name="create-documentation-from-tests"></a>Create Documentation from Tests

The test checks all what we said about at the beginning of the chapter. So if you are able to write well test-specification, we can assume a good idea to create documentation based on them. We will use gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) for this purpose (gem was already added into `Gemfile`).

We should rewrite tests into supposed format using `rspec_api_documentation` DSL and put them into appropriate folder - `spec/acceptance`. We should configure `rspec_api_documentation`. And we should create `rake` tasks for generating documentation. Let's do this.

Create please folder `spec/acceptance` and add file `service_spec.rb` in it. This is same tests but written using `rspec_api_documentation` DSL so gem can generate documentation based on tests.

```ruby
require "spec_helper"

resource "Users" do
  get "/api/v1/users/me.json" do
    example "retrieve admin user JSON representation of provided token of admin user" do
      header "Authorization", "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
      do_request
      expect(status).to eq 200
      expect(response_body).to eq '{"user":{"type":"AdminUser"}}'
    end

    example "retrieve regular user JSON representation of provided token of regular user" do
      header "Authorization", "OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"
      do_request
      expect(status).to eq 200
      expect(response_body).to eq '{"user":{"type":"RegularUser"}}'
    end

    example "respond with 401 status and JSON error message if access token expired or incorrect" do
      header "Authorization", "OAuth 7564e5ab2d46d5af38e99e5490eea2c86b96f6a638d77fa0b124125ed26347eb"
      do_request
      expect(status).to eq 401
      expect(response_body).to eq '{"message":"Invalid or expired token"}'
    end

    example_request "responds with 403 status and JSON error message if access token not provided" do
      expect(status).to eq 403
      expect(response_body).to eq '{"message":"Access Forbidden"}'
    end
  end
end
``` 

Add please gem configuration into file `spec/spec_helper.rb`

```ruby
require_relative "../service"
require "rack/test"

RSpec.configure do |config|
  config.include Rack::Test::Methods

  def app
    Sinatra::Application
  end
end

require "rspec_api_documentation/dsl"

RspecApiDocumentation.configure do |config|
  config.docs_dir = Pathname.new(Sinatra::Application.root).join("doc")
  config.app = Sinatra::Application
  config.api_name = "Users API"
  config.format = :html
end
```

Now you can delete file `spec/service_spec.rb` and use instead it file `spec/acceptance/service_spec.rb`.

Create file `Rakefile` in folder `users`.

```ruby
require_relative 'service'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

We don't need tasks for generating documentation in `production` environment, so we don't load it in Rakefile in this case.

You can see all available `rake` tasks by running `rake -T`

    $ rake -T

    rake docs:generate          # Generate API request documentation from API specs
    rake docs:generate:ordered  # Generate API request documentation from API specs (ordered)

And we can generate documentation with either of this two tasks

    $ rake docs:generate

Documentation in HTML format will be saved under folder `doc`. You should use test examples names to be more suitable for humans (programmers that will be use your service).

## <a name="exclude-specific-test-examples-from-documentation"></a>Exclude Certain Test Examples from Documentation

You can exclude certain test cases from the documentation with the option `document: nil`, for example:

```ruby
# This example does not fall into the documentation.
example_request "no token - no user", document: nil do
  expect(status).to eq 403
  expect(response_body).to eq '{"message":"Access Forbidden"}'
end
```

## <a name="summary"></a>Summary

We used gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) for creating auto-generated documentation based on rspec tests. Sometimes it helps.
