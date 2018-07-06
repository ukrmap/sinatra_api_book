Кіраўнік #3. Апрацоўка памылак і лагаванне
==========================================

У гэтай частцы мы створым просты вэб-сэрвіс для цэлалікавага дзялення натуральных лікаў. Сэрвіс павінен быць у стане апрацоўваць непрадбачаныя памылкі (вы, напэўна, здагадаліся, што гэта будзе дзяленне на нуль). І мы павінны быць у стане выкарыстаць логгер для адладкі і логгирования кожнага звароту да сэрвісу. Гэта будзе лёгка.

## <a name="infrastructure"></a>Інфраструктура

Калі ласка, стварыце тэчку `divisor` для нашага сэрвісу і файл `Gemfile` ў ёй

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

Затым перайдзіце да гэтай тэчцы ў тэрмінале і выканайце

    $ bundle install

Стварыце, калі ласка, тэчку `log` ў тэчцы `divisor`. Гэта тэчка, у якой мы будзем захоўваць файлы логаваў для розных асяроддзяў (`development.log`, `test.log`, `production.log`).

## <a name="basic-implementation"></a>Базавая рэалізацыя

Стварыце, калі ласка, файл `service.rb` ўнутры тэчкі `divisor`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

get "/api/v1/ratio/:a/:b" do
   content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Цяпер мы можам запусціць наш сэрвіс.

    $ ruby service.rb
    [2015-01-30 13:51:25] INFO  WEBrick 1.3.1
    [2015-01-30 13:51:25] INFO  ruby 2.0.0 (2014-02-24) [x86_64-darwin12.5.0]
    == Sinatra/1.4.5 has taken the stage on 4567 for development with backup from WEBrick
    [2015-01-30 13:51:25] INFO  WEBrick::HTTPServer#start: pid=72295 port=4567

І мы можам выкарыстоўваць наш сэрвіс для вылічэнні выніку цэлалікавага дзялення двух цэлых лікаў. Перайдзіце ў іншае акно (ці ўкладку) тэрмінала і запусціўшы каманду:

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

Вэб-сэрвіс працуе. Мы выкарыстоўвалі сцяг `-i`, каб убачыць код стану HTTP адказу і загалоўкі (headers). Мы можам стварыць `rspec` тэст. Калі ласка, стварыце тэчку `spec` і файл `service_spec.rb` ў тэчцы `spec`.

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

Радок `expect(last_response).to be_ok` гэта скарачэнне для `expect(last_response.status).to eq 200` (HTTP код 200 азначае адказ ОК - усё ў парадку).

## <a name="ordeal-web-service"></a>Выпрабаванне вэб-сэрвісу

Ну, мы чакалі гэтага з пачатку кіраўніка, давайце дзяліць на нуль.

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

Памылка чакалася, але чаму мы бачым `ruby` выключэнне ў HTTP адказе? Гэта паводзіны `sinatra` ў `development` рэжыме па змаўчанні, каб пазбегнуць гэтага, мы можам адключыць наладу `show_exceptions` ў `service.rb`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Перазапусціце сэрвіс і зноў выканайце

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

Мы можам змяніць паведамленне пра памылку.

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

Перазапусціць сэрвіс яшчэ раз і зноў выканайце

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

Гэта тое, што будзе адбывацца ў `production` калі адбываецца раптоўная памылка. Давайце таксама праверым гэта з дапамогай `rspec`.

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

Дадайце яшчэ адзін параметр для асяроддзя `test`

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

Запускаем тэсты

    $ rspec
    ..

    Finished in 0.05946 seconds (files took 0.67036 seconds to load)
    2 examples, 0 failures

Цяпер мы ведаем, што карыстальнік паведамляецца належным чынам, калі адбываецца збой. Але мы таксама хочам ведаць пра ўзнікненне памылак.

## <a name="logging-request-params-and-http-response-code"></a>Лагаванне параметраў запыту і кода стану HTTP адказу

Для лагаванне доступу да сэрвісу мы будзем выкарыстоўваць `Rack :: CommonLogger`, гэта адзін з карысных` middleware`, якія поставляеются разам з `rack`. Паглядзіце на абноўлены `service.rb`:

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

Зараз усе запыты да сэрвісу будуць рэгістравацца ў log-файле для бягучага акружэння. Вось змесціва `test.log` пасля запуску тэстаў:

    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0086
    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

Фармат запісу - адна радок для запыту: IP-адрас, бягучы ідэнтыфікатар карыстальніка (пуста), час запыту, URL-шлях з метадам HTTP, HTTP код адказу, даўжыня цела адказу (колькасць знакаў) і падчас выканання ў секундах.

## <a name="custom-logging"></a>Выкарыстанне Logger ў кодзе

Часам нам трэба выкарыстоўваць `logger` для розных мэтаў у URL-маршруце або ў любым іншым кодзе прыкладання. Мы будзем выкарыстоўваць той жа `log` файл для гэтага, але вы можаце выкарыстаць іншы. Абноўлены `service.rb`:

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

Калі мы запусцім тэсты зноў новая інфармацыя адладкі будзе захоўвацца ў файле `test.log`.

    compute the result of integer division 23 / 4
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0054
    compute the result of integer division 1 / 0
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

## <a name="email-notifications-about-errors"></a>E-mail паведамлення аб памылках

Калі сэрвіс працуе ў `production` мы павінны быць апавешчаныя аб непрадбачаных памылках адразу, напрыклад па электроннай пошце. Мы будзем выкарыстоўваць gem [rusen](https://github.com/Moove-it/rusen) для гэтага. Мы будзем адпраўляць электронную пошту толькі `if settings.production?` Або выкарыстоўваць розныя конфігі. Абноўлены `service.rb`:

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

Дадайце, калі ласка, файл `rusen_config.rb` ў папку` divisor`.

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

Мы атрымаем лісты з апісаннем памылкі, трасіроўку (backtrace) і зменныя асяроддзі ў `production`. Тая ж інфармацыя адлюстроўваецца ў кансолі ў `development` і `test` рэжымах (толькі для праверачных мэтаў).

## <a name="airbrake"></a>Airbrake

Вы таксама можаце інтэграваць дадатак з сэрвісам для кантролю памылак, такім як [Airbrake](https://airbrake.io/). Для гэтага вам неабходна мець ключ API ад зарэгістраванай ўліковага запісу на Airbrake. Ці вы можаце выкарыстоўваць gem ад Airbrake з адкрытым зыходным кодам [errbit](https://github.com/errbit/errbit), каб наладзіць сервер адсочванне памылак самастойна. Інтэграваць сэрвіс з дадаткам не сложно: дадайце gem `airbrake` ў` Gemfile` і выканайце `bundle install`, затым дадайце налады ў` service.rb`:

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

Вось і ўсё.

## <a name="summary"></a>Рэзюмэ

У гэтай частцы мы зрабілі кароткі агляд, налады лагаванне ў `sinatra` і налады апавяшчэнняў аб памылках па электроннай пошце выкарыстоўваючы gem [rusen](https://github.com/Moove-it/rusen). Існуе падобны gem для `rails` - [exception_notification](https://github.com/smartinez87/exception_notification). Мы таксама вельмі коратка абмеркавалі інтэграцыю з сэрвісам адсочвання памылак на прыкладзе `Airbrake`.
