Глава #3. Обробка помилок та логування
======================================

У цій главі ми створимо простий веб-сервіс для цілочисельного ділення натуральних чисел. Сервіс повинен бути в змозі обробляти непередбачені помилки (ви, напевно, здогадалися, що це буде ділення на нуль). І ми повинні бути в змозі використати логгер для налагодження (debug) та логування кожного звернення до сервісу. Це буде легко.

## <a name="infrastructure"></a>Інфраструктура

Будь ласка, створіть теку `divisor` для нашого сервісу і файл` Gemfile` в ній

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

Потім перейдіть до цієї папки в терміналі і виконайте

    $ bundle install

Створіть, будь ласка, теку `log` в теці `divisor`. Це папка, в якій ми будемо зберігати файли логів для різних оточень (`development.log`, `test.log`, `production.log`).

## <a name="basic-implementation"></a>Базова реалізація

Створіть, будь ласка, файл `service.rb` всередині папки `divisor`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

get "/api/v1/ratio/:a/:b" do
   content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Тепер ми можемо запустити наш сервіс.

    $ ruby service.rb
    [2015-01-30 13:51:25] INFO  WEBrick 1.3.1
    [2015-01-30 13:51:25] INFO  ruby 2.0.0 (2014-02-24) [x86_64-darwin12.5.0]
    == Sinatra/1.4.5 has taken the stage on 4567 for development with backup from WEBrick
    [2015-01-30 13:51:25] INFO  WEBrick::HTTPServer#start: pid=72295 port=4567

І ми можемо використовувати наш сервіс для обчислення результату цілочисельного ділення двох цілих чисел. Перейдіть в інше вікно (або вкладку) терміналу і виконайте:

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

Веб-сервіс працює. Ми використовували опцію `-i`, щоб побачити код стану HTTP відповіді і заголовки (headers). Ми можемо створити `rspec` тест. Будь ласка, створіть теку `spec` і файл` service_spec.rb` в теці `spec`.

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

Рядок `expect(last_response).to be_ok` це скорочення для `expect(last_response.status).to eq 200` (HTTP код 200 означає відповідь ОК - все в порядку).

## <a name="ordeal-web-service"></a>Випробування веб-сервісу

Ну, ми чекали цього з початку глави, давайте ділити на нуль.

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

Помилка очікувалася, але чому ми бачимо `ruby` exception у HTTP відповіді? Це поведінка `sinatra` в `development` режимі за замовчуванням, щоб уникнути цього, ми можемо відключити настройку `show_exceptions` в `service.rb`.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym

disable :show_exceptions

get "/api/v1/ratio/:a/:b" do
  content_type :txt
  "#{params[:a].to_i / params[:b].to_i}"
end
```

Перезапустіть сервіс і знову виконайте

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

Ми можемо змінити повідомлення про помилку.

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

Перезапустити сервіс ще раз і знову виконайте

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

Це те, що буде відбуватися в `production` якщо відбувається непередбачена помилка. Давайте також перевіримо це за допомогою `rspec`.

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

Додайте ще один параметр для оточення `test`

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

Запускаємо тести

    $ rspec
    ..

    Finished in 0.05946 seconds (files took 0.67036 seconds to load)
    2 examples, 0 failures

Тепер ми знаємо, що користувач повідомляється належним чином, якщо відбувається збій. Але ми також хочемо знати про виникнення помилок.

## <a name="logging-request-params-and-http-response-code"></a>Логування параметрів запиту та коду стану HTTP відповіді

Для логування доступу до сервісу ми будемо використовувати `Rack :: CommonLogger`, це один з корисних` middleware`, які поставляеются разом з `rack`. Подивіться на оновлений `service.rb`:

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

Тепер всі запити до сервісу будуть реєструватися в log-файлі для поточного оточення. Ось вміст `test.log` після запуску тестів:

    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0086
    127.0.0.1 - - [01/Feb/2015:19:57:36 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

Формат запису - одна рядок для запиту: IP-адреса, поточний ідентифікатор користувача (порожньо), час запиту, URL-шлях з методом HTTP, HTTP код відповіді, довжина тіла відповіді (кількість символів) і час виконання в секундах.

## <a name="custom-logging"></a>Використання Logger в коді

Іноді нам потрібно використовувати `logger` для різних цілей в URL-маршруті або в будь-якому іншому коді програми. Ми будемо використовувати той же `log` файл для цього, але ви можете використовувати інший. Оновлений `service.rb`:

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

Якщо ми запустимо тести знову нова інформація буде збережена у файлі `test.log`.

    compute the result of integer division 23 / 4
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/23/4 " 200 1 0.0054
    compute the result of integer division 1 / 0
    127.0.0.1 - - [01/Feb/2015:20:04:34 +0200] "GET /api/v1/ratio/1/0 " 500 58 0.0006

## <a name="email-notifications-about-errors"></a>E-mail повідомлення про помилки

Коли сервіс працює в `production` ми повинні бути повідомлені про непередбачені помилки відразу, наприклад по електронній пошті. Ми будемо використовувати gem [rusen](https://github.com/Moove-it/rusen) для цього. Ми будемо відправляти електронну пошту тільки `if settings.production?` або використовувати різні конфіги. Оновлений `service.rb`:

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

Додайте, будь ласка, файл `rusen_config.rb` в теку` divisor`.

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

Ми отримаємо листи з описом помилки, трасування (backtrace) і змінні оточення в `production`. Та ж інформація відображається в консолі в `development` і `test` режимах (тільки для перевірочних цілей).

## <a name="airbrake"></a>Airbrake

Ви також можете інтегрувати додаток з сервісом для відслідковування помилок, таким як [Airbrake](https://airbrake.io/). Для цього вам необхідно мати ключ API від зареєстрованого облікового запису на Airbrake. Або ви можете використовувати gem від Airbrake з відкритим вихідним кодом [errbit](https://github.com/errbit/errbit), щоб налаштувати сервер відстеження помилок самостійно. Інтегрувати сервіс з додатком не складно: додайте gem `airbrake` в `Gemfile` і виконайте `bundle install`, потім додайте настройки в `service.rb`:

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

Ось і все.

## <a name="summary"></a>Резюме

У цій главі ми зробили короткий огляд, налаштування логування в `sinatra` і налаштування повідомлень про помилки по електронній пошті використовуючи gem [rusen](https://github.com/Moove-it/rusen). Існує схожий gem для `rails` - [exception_notification](https://github.com/smartinez87/exception_notification). Ми також дуже коротко обговорили інтеграцію з сервісом відслідковування помилок на прикладі `Airbrake`.
