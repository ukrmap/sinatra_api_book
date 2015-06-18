Chapter #3. Error Handling and Logging
======================================

In this chapter, we will create a simple web service for integer division of natural numbers. Service must be able to handle unexpected errors (you might have guessed that it would be division by zero). And we should be able to use the logger to log the debug and every call to service. It will be easy.

## <a name="infrastructure"></a>Infrastructure

Please create folder `divisor` for our service and file `Gemfile` in it

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'rusen'
gem 'pony'

group :test do
  gem 'rspec'
  gem 'rack-test'
end
```

Then navigate to this folder in terminal and run

    $ bundle install

Create please folder `log` inside `divisor`. This is the folder in which we will store the log files for different environments (`development.log`, `test.log`, `production.log`).

## <a name="basic-implementation"></a>Basic implementation

Create please file `service.rb` inside folder `divisor`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

get "/api/v1/ratio/:a/:b" do
   content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Now we can run our service.

    $ ruby service.rb
    [2015-01-30 13:51:25] INFO  WEBrick 1.3.1
    [2015-01-30 13:51:25] INFO  ruby 2.0.0 (2014-02-24) [x86_64-darwin12.5.0]
    == Sinatra/1.4.5 has taken the stage on 4567 for development with backup from WEBrick
    [2015-01-30 13:51:25] INFO  WEBrick::HTTPServer#start: pid=72295 port=4567

And we can use our service to compute the result of integer division of two integers. Navigate in other window (or tab) of terminal and run:

    $ curl -i -X GET "localhost:4567/api/v1/ratio/23/4"
    HTTP/1.1 200 OK 
    Content-Type: text/html;charset=utf-8
    Content-Length: 1
    X-Xss-Protection: 1; mode=block
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    Server: WEBrick/1.3.1 (Ruby/2.0.0/2014-02-24)
    Date: Fri, 30 Jan 2015 11:54:16 GMT
    Connection: Keep-Alive

    5

Web-service works. We have used flag `-i` to see HTTP status code of response and headers. We can create `rspec` test. Please create folder `spec` and file `service_spec.rb` in folder `spec`.

```ruby
ENV['RACK_ENV'] = 'test'
require_relative "../service"

RSpec.configure do |config|
  config.include Rack::Test::Methods

  def app
    Sinatra::Application
  end
end

describe "Divisor Service" do
  describe "GET /api/v1/ratio/:a/:b" do
    it "computes the result of integer division of two integers" do
      get "/api/v1/ratio/23/4"
      expect(last_response).to be_ok
      expect(last_response.body).to eq "5"
    end
  end
end
```

Line `expect(last_response).to be_ok` is short for `expect(last_response.status).to eq 200` (HTTP code 200 means OK response).

## <a name="ordeal-web-service"></a>Ordeal web service

Well, we've been waiting for this since the beginning of the chapter, let's divide by zero.

    $ curl -i -X GET "localhost:4567/api/v1/ratio/1/0"
    HTTP/1.1 500 Internal Server Error 
    Content-Type: text/plain
    Content-Length: 4563
    Server: WEBrick/1.3.1 (Ruby/2.0.0/2014-02-24)
    Date: Fri, 30 Jan 2015 12:03:43 GMT
    Connection: Keep-Alive

    ZeroDivisionError: divided by 0
      service.rb:6:in `/'
      service.rb:6:in `block in <main>'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1603:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1603:in `block in compile!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:966:in `[]'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:966:in `block (3 levels) in route!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:985:in `route_eval'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:966:in `block (2 levels) in route!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1006:in `block in process_route'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1004:in `catch'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1004:in `process_route'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:964:in `block in route!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:963:in `each'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:963:in `route!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1076:in `block in dispatch!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `block in invoke'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `catch'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `invoke'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1073:in `dispatch!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:898:in `block in call!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `block in invoke'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `catch'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1058:in `invoke'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:898:in `call!'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:886:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/xss_header.rb:18:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/path_traversal.rb:16:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/json_csrf.rb:18:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/base.rb:49:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/base.rb:49:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-protection-1.5.3/lib/rack/protection/frame_options.rb:31:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-1.6.0/lib/rack/logger.rb:15:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-1.6.0/lib/rack/commonlogger.rb:33:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:217:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:210:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-1.6.0/lib/rack/head.rb:13:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-1.6.0/lib/rack/methodoverride.rb:22:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/show_exceptions.rb:21:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:180:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:2014:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1478:in `block in call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1788:in `synchronize'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/sinatra-1.4.5/lib/sinatra/base.rb:1478:in `call'
      /Users/alex/.rvm/gems/ruby-2.0.0-p451/gems/rack-1.6.0/lib/rack/handler/webrick.rb:89:in `service'
      /Users/alex/.rvm/rubies/ruby-2.0.0-p451/lib/ruby/2.0.0/webrick/httpserver.rb:138:in `service'
      /Users/alex/.rvm/rubies/ruby-2.0.0-p451/lib/ruby/2.0.0/webrick/httpserver.rb:94:in `run'
      /Users/alex/.rvm/rubies/ruby-2.0.0-p451/lib/ruby/2.0.0/webrick/server.rb:295:in `block in start_thread'

Error was expected, but why we see `ruby` exception in HTTP response? This is behaviour of `sinatra` in `development` mode by default, to avoid this we can disable setting `show_exceptions` in `service.rb`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Restart service and run again

    $ curl -i -X GET "localhost:4567/api/v1/ratio/1/0"
    HTTP/1.1 500 Internal Server Error 
    Content-Type: text/html;charset=utf-8
    Content-Length: 30
    X-Xss-Protection: 1; mode=block
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    Server: WEBrick/1.3.1 (Ruby/2.0.0/2014-02-24)
    Date: Fri, 30 Jan 2015 12:17:57 GMT
    Connection: Keep-Alive

    <h1>Internal Server Error</h1>

We can customise error message.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

error do
  "An internal server error occurred. Please try again later."
end
```

Restart service one more time and run

    # curl -i -X GET "localhost:4567/api/v1/ratio/1/0"
    HTTP/1.1 500 Internal Server Error 
    Content-Type: text/html;charset=utf-8
    Content-Length: 58
    X-Xss-Protection: 1; mode=block
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    Server: WEBrick/1.3.1 (Ruby/2.0.0/2014-02-24)
    Date: Fri, 30 Jan 2015 12:20:00 GMT
    Connection: Keep-Alive

    An internal server error occurred. Please try again later.

This what will happen in `production` if unexpected error occurs. Let's also test this with `rspec`.

```ruby
ENV['RACK_ENV'] = 'test'
require_relative "../service"

RSpec.configure do |config|
  config.include Rack::Test::Methods

  def app
    Sinatra::Application
  end
end

describe "Divisor Service" do
  describe "GET /api/v1/ratio/:a/:b" do
    it "computes the result of integer division of two integers" do
      get "/api/v1/ratio/23/4"
      expect(last_response).to be_ok
      expect(last_response.body).to eq "5"
    end

    it "handles unexpected errors" do
      get "/api/v1/ratio/1/0"
      expect(last_response.status).to eq 500
      expect(last_response.body).to eq "An internal server error occurred. Please try again later."
    end
  end
end
```

Add one more setting for `test` environment

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions # in production it is false, so you probably do not need it
disable :raise_errors # in production and dev mode it is false, so you probably do not need it

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

error do
  "An internal server error occurred. Please try again later."
end
```

Run tests

    $ rspec
    ..

    Finished in 0.05946 seconds (files took 0.67036 seconds to load)
    2 examples, 0 failures

Now we know that user is notified properly if service crashes. But we also want to know about occurrence of errors.

## <a name="logging-request-params-and-http-response-code"></a>Logging request parameters and HTTP response code

For the logging the access to the service we will use `Rack::CommonLogger`, this is one of useful `middleware` distributed with `rack`. Look for an updated `service.rb`:

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions # in production it is false, so you probably do not need it
disable :raise_errors # in production and dev mode it is false, so you probably do not need it

# Logging request params and HTTP response code
require 'logger'
set :root, File.dirname(__FILE__)
log_file = File.new(File.join(settings.root, 'log', "#{settings.environment}.log"), 'a+')
log_file.sync = true
use Rack::CommonLogger, log_file

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

error do
  "An internal server error occurred. Please try again later."
end
```

Now all service requests will be recorded in the log-file for the current environment. Here are the contents of `test.log` after running the tests:

    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0086
    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

Recording format - one line for the request: IP address, current user ID (empty), request time, route with HTTP method, response HTTP code, response body length (number of characters) and execution time in seconds.

## <a name="custom-logging"></a>Custom Logging (Using logger in code)

Sometimes we need use `logger` for different purposes in route or in any other application code. We will use the same `log` file for this, but you can use other one. Updated `service.rb`:

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions # in production it is false, so you probably do not need it
disable :raise_errors # in production and dev mode it is false, so you probably do not need it

# Logging request params and HTTP response code
require 'logger'
set :root, File.dirname(__FILE__)
log_file = File.new(File.join(settings.root, 'log', "#{settings.environment}.log"), 'a+')
log_file.sync = true
use Rack::CommonLogger, log_file

# Custom logging
logger = Logger.new(log_file)
logger.formatter = ->(severity, time, progname, msg) { "#{msg}\n" }
before { env['rack.logger'] = logger }

get "/api/v1/ratio/:a/:b" do
  logger.info "compute the result of integer division #{params[:a]} / #{params[:b]}"
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

error do
  "An internal server error occurred. Please try again later."
end
```

If we run tests again new debugging info will be saved in the `test.log`.

    compute the result of integer division 23 / 4
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0054
    compute the result of integer division 1 / 0
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

## <a name="email-notifications-about-errors"></a>E-mail notifications about errors

When service is running on `production` we need to be notified about unexpected errors immediately, for example through e-mail. We will use gem [rusen](https://github.com/Moove-it/rusen) for this. We will send emails only `if settings.production?` or use different configs. Updated `service.rb`:

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions # in production it is false, so you probably do not need it
disable :raise_errors # in production and dev mode it is false, so you probably do not need it

# Logging request params and HTTP response code
require 'logger'
set :root, File.dirname(__FILE__)
log_file = File.new(File.join(settings.root, 'log', "#{settings.environment}.log"), 'a+')
log_file.sync = true
use Rack::CommonLogger, log_file

# Custom logging
logger = Logger.new(log_file)
logger.formatter = ->(severity, time, progname, msg) { "#{msg}\n" }
before { env['rack.logger'] = logger }

get "/api/v1/ratio/:a/:b" do
  logger.info "compute the result of integer division #{params[:a]} / #{params[:b]}"
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

require_relative 'rusen_config'
error do
  # Arguments are: exception, request, environment, session
  Rusen.notify(env['sinatra.error'], {}, env, {})
  "An internal server error occurred. Please try again later."
end
```

Add please file `rusen_config.rb` into folder `divisor`.

```ruby
configure :production do
  Rusen.settings.outputs = [:pony]
  Rusen.settings.sections = [:backtrace, :environment]
  Rusen.settings.email_prefix = "[ERROR Divisor API] "
  Rusen.settings.sender_address = "your-email@gmail.com"
  Rusen.settings.exception_recipients = %w(your-email@gmail.com)
  Rusen.settings.smtp_settings = {
    address: "smtp.gmail.com",
    port: 587,
    domain: "mail.google.com",
    authentication: :plain,
    user_name: "your-email@gmail.com",
    password: "xxxxxxxx",
    enable_starttls_auto: true
  }
end

configure :development, :test do
  Rusen.settings.outputs = [:io]
  Rusen.settings.sections = [:backtrace, :environment]
end
```

We will get emails with error description, backtrace and environment variables in `production` and same information is displayed in console in `development` and `test` mode (only for debug purposes).

## <a name="airbrake"></a>Airbrake

You also can integrate with error tracking service like [Airbrake](https://airbrake.io/). To do this, you must have an API key from a registered account on Airbrake. Or you can use Airbrake's open source gem [errbit](https://github.com/errbit/errbit) to set up error tracking server yourself. Integrate tracking server with service is easy: add gem `airbrake` into `Gemfile` and run `bundle install`, then add configs into `service.rb`:

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions # in production it is false, so you probably do not need it
disable :raise_errors # in production and dev mode it is false, so you probably do not need it

# Logging request params and HTTP response code
require 'logger'
set :root, File.dirname(__FILE__)
log_file = File.new(File.join(settings.root, 'log', "#{settings.environment}.log"), 'a+')
log_file.sync = true
use Rack::CommonLogger, log_file

# Custom logging
logger = Logger.new(log_file)
logger.formatter = ->(severity, time, progname, msg) { "#{msg}\n" }
before { env['rack.logger'] = logger }

get "/api/v1/ratio/:a/:b" do
  logger.info "compute the result of integer division #{params[:a]} / #{params[:b]}"
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end

error do
  "An internal server error occurred. Please try again later."
end

configure :production do
  Airbrake.configure do |config|
    config.api_key = 'your_api_key'
    config.environment_name = 'Divisor API'
  end
  use Airbrake::Sinatra
end
```

That is it.

## <a name="summary"></a>Summary

In this chapter we did short overview how to set up logging in `sinatra` and how to use e-mail exception notification using gem [rusen](https://github.com/Moove-it/rusen). There is similar gem for `rails` - [exception_notification](https://github.com/smartinez87/exception_notification). We are also very briefly discussed the integration with bug tracking service for example `Airbrake`.
