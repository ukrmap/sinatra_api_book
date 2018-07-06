Глава #4. Документация и тестирование
=====================================

Если вы создаёте API вам также необходимо предоставить документацию (спецификации) для него. И вам также необходимо создавать тесты, которые являются гарантией надежности, также как и спецификацией. В этой главе мы постараемся объединить документацию и тесты.

Мы создадим минималистское внутри API (users service), которое мы будем использовать в других серфисах для аутентификации.

## <a name="planning-interface"></a>Планирование интерфейса

Users API будет иметь один URI-путь, который будет возвращать JSON-представление конкретного пользователя. Пользователь (может быть другой веб-сервис) нашего API должен передать правильный ключ аутентификации в HTTP заголовке запроса. Ключ (token) проверки подлинности - это некоторая последовательность символов - строка, которая известна, только пользователю и веб-сервису. В реальности, если ключ истек, пользователь должен запросить новый, но мы не будем создаавать этот фунцционал и будем использовать вечные ключи для простоты.

Мы будем использовать пользователей двух типов: "AdminUser" и "RegularUser" (на самом деле только один "AdminUser" и один "RegularUser").

Пример запроса к сервису:

    $ curl -X GET "localhost:4567/api/v1/users/me.json" -H "Authorization: OAuth user_token"

    {"user":{"type":"RegularUser"}}

Где строка `user_token` должна быть заменена актуальным токеном пользователя. Токен - это значение HTTP заголовка "Authorization" с префиксом "OAuth" (для [OAuth](https://ru.wikipedia.org/wiki/OAuth) или [OAuth 2.0](http://oauth.net/2/)).
Если токен не является правильным сервис должен вернуть HTTP ответ c кодом 401 (Unauthorized) и соответствующим сообщением об ошибке в формате JSON. Если токен не передан сервис должен вернуть HTTP ответ с кодом 403 (Forbidden) и также соответствующим сообщением об ошибке в формате JSON.

## <a name="create-service"></a>Создание сервиса

Создайте, пожалуйста, папку `users` где-то в системе, это будет корневая папка для нового сервиса. Создайте в ней файл `Gemfile` со следующем содержимым:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'rspec'
gem 'rack-test'
gem 'rspec_api_documentation'
```

Зайдите в папку `users` в терминале и выполните:

    $ bundle install

Создайте файл `service.rb` в папке `users` со следующим содержимым:

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

Мы добавили один URL-маршрут (с блоком). Блок содержит значение двух токенов (которые, очевидно, не истекут) непосредственно в коде. Он возвращает пользователя типа "AdminUser" или "RegularUser" в зависимости от токена. Мы можем запустить сервис с помощью команды `ruby service.rb` и протестировать его (из другой вкладки или окна терминала):

    $ curl -X GET "localhost:4567/api/v1/users/me.json" \
      -H "Authorization: OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"

    {"user":{"type":"RegularUser"}}

Мы разделили `curl` запрос на две строки символом "\" (обратная косая черта).

Мы также можем проверить ответ сервиса, если передан неверный токен или токен не передан. Используйте `-i` флаг, чтобы увидеть HTTP ответ с заголовками.

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

## <a name="create-tests"></a>Создание тестов

Создайте, пожалуйста, папку `spec` в папке `users` и файл `service_spec.rb` в папке `spec` со следующем содержимым:

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

И файл `spec_helper.rb` (также в папке `spec`)

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

Теперь мы можем запустить наши тесты:

    $ rspec
    ....

    Finished in 0.06302 seconds (files took 0.3102 seconds to load)
    4 examples, 0 failures

## <a name="create-documentation-from-tests"></a>Создание документации на основе тестов

Тест проверяет все то, о чем мы говорили в начале главы. Так что, если вы в состоянии хорошо писать тесты-спецификации, то можно считать хорошей идеей создание документации из них. Мы будем использовать gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для этой цели (gem был уже добавлен в `Gemfile`).

Мы должны переписать тесты в предполагаемой форме с использованием `rspec_api_documentation` DSL и поместить их в соответствующую папку - `spec/acceptance`. Мы должны настроить `rspec_api_documentation`. И мы должны создать `rake` задачи для создания документации. Давайте сделаем это.

Создайте, пожалуйста, папку `rspec/acceptance` и добавьте в неё файл `service_spec.rb`. Это те же самые тесты, но написаные с использованием `rspec_api_documentation` DSL, таким образом gem может генерировать документацию на основании тестов.

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

Добавьте, пожалуйста, конфигурацию для гема в файл `spec/spec_helper.rb`

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

Теперь вы можете удалить файл `spec/service_spec.rb` и использовать вместо него файл `spec/acceptance/service_spec.rb`.

Создайте файл `Rakefile` в папке `users`.

```ruby
require_relative 'service'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Нам не нужны задачи для создания документации в `production` среде, поэтому мы не загружаем их в Rakefile в этом случае.

Вы можете увидеть все доступные `rake` задачи с помощью команды `rake -T`

    $ rake -T

    rake docs:generate          # Generate API request documentation from API specs
    rake docs:generate:ordered  # Generate API request documentation from API specs (ordered)

И мы можем сгенерировать документацию с помощью любой из этих двух задач

    $ rake docs:generate

Документация в формате HTML будет сохранена в папке `doc`. Вам стоит использовать имена для тестов более подходящие для людей (программистов, которые будут пользоваться вашим сервисом).

## <a name="exclude-specific-test-examples-from-documentation"></a>Исключение некоторых тестов из документации

Вы можете исключить определенные тестовые примеры из документации с помощью опции `document: false`, например:

```ruby
# This example does not fall into the documentation.
example_request "no token - no user", document: false do
  expect(status).to eq 403
  expect(response_body).to eq '{"message":"Access Forbidden"}'
end
```

## <a name="summary"></a>Резюме

Мы использовали gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для создания автоматически сгенерированной документации на основе RSpec тестов. Иногда это может быть полезным.
