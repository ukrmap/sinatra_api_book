Глава #4. Документація і тестування
===================================

Якщо ви створюєте API вам також необхідно надати документацію (специфікації) для нього. І вам також необхідно створювати тести, які є гарантією надійності, також як і специфікацією. У цій главі ми постараємося об'єднати документацію і тести.

Ми створимо мінімалістське всередині API (users service), яке ми будемо використовувати в інших серфісах для аутентифікації.

## <a name="planning-interface"></a>Планування інтерфейсу

Users API матиме один URI-шлях, який буде повертати JSON-уявлення конкретного користувача. Користувач (може бути інший веб-сервіс) нашого API повинен передати правильний ключ аутентифікації в HTTP заголовку запиту. Ключ (token) перевірки автентичності - це деяка послідовність символів - рядок, який відомий, тільки користувачеві і веб-сервісу. У реальності, якщо термін використання ключа минув, користувач повинен запросити новий, але ми не будемо создаавать цей фунцціонал і будемо використовувати вічні ключі для простоти.

Ми будемо використовувати користувачів двох типів: "AdminUser" і "RegularUser" (насправді тільки один "AdminUser" і один "RegularUser").

Приклад запиту до сервісу:

    $ curl -X GET "localhost:4567/api/v1/users/me.json" -H "Authorization: OAuth user_token"

    {"user":{"type":"RegularUser"}}

Де рядок `user_token` повинен бути замінений актуальним токеном користувача. Токен - це значення HTTP заголовка "Authorization" з префіксом "OAuth" (для [OAuth](https://ru.wikipedia.org/wiki/OAuth) або [OAuth 2.0](http://oauth.net/2/)).
Якщо токен не є правильним сервіс повинен повернути HTTP відповідь c кодом 401 (Unauthorized) і відповідним повідомленням про помилку в форматі JSON. Якщо токен не переданий сервіс повинен повернути HTTP відповідь з кодом 403 (Forbidden) і також відповідним повідомленням про помилку в форматі JSON.

## <a name="create-service"></a>Створення сервісу

Створіть, будь ласка, теку `users` десь в системі, це буде коренева тека для нового сервісу. Створіть в ній файл `Gemfile` наступного вмісту:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'rspec'
gem 'rack-test'
gem 'rspec_api_documentation'
```

Зайдіть в теку `users` в терміналі і виконайте:

    $ bundle install

Створіть файл `service.rb` в теці `users` з наступним вмістом:

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

Ми додали один URL-маршрут (з блоком). Блок містить значення двох токенов (час використання яких, очевидно, не мине) безпосередньо в коді. Він повертає користувача типу "AdminUser" або "RegularUser" залежно від токена. Ми можемо запустити сервіс за допомогою команди `ruby service.rb` і протестувати його (з іншої вкладки або вікна терміналу):

    $ curl -X GET "localhost:4567/api/v1/users/me.json" \
      -H "Authorization: OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"

    {"user":{"type":"RegularUser"}}

Ми розділили `curl` запит на два рядки символом "\" (зворотна коса риска).

Ми також можемо перевірити відповідь сервісу, якщо передано невірний токен або токен не передано. Використовуйте опцію `-i`, щоб побачити HTTP відповідь з заголовками.

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

## <a name="create-tests"></a>Створення тестів

Створіть, будь ласка, теку `spec` в теці `users` і файл `service_spec.rb` в теці `spec` наступного вмісту:

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

І файл `spec_helper.rb` (також в теці `spec`)

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

Тепер ми можемо запустити наші тести:

    $ rspec
    ....

    Finished in 0.06302 seconds (files took 0.3102 seconds to load)
    4 examples, 0 failures

## <a name="create-documentation-from-tests"></a>Создание документации на основе тестов

Тест перевіряє все те, про що ми говорили на початку глави. Так що, якщо ви в змозі добре писати тести-специфікації, то можна вважати гарною ідеєю створення документації з них. Ми будемо використовувати gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для цієї мети (gem був уже доданий до `Gemfile`).

Ми повинні переписати тести в необхідну форму з використанням `rspec_api_documentation` DSL і помістити їх у відповідну теку - `spec/acceptance`. Ми повинні налаштувати `rspec_api_documentation`. І ми повинні створити `rake` завдачі для створення документації. Давайте зробимо це.

Створіть, будь ласка, теку `rspec/acceptance` і додайте до неї файл `service_spec.rb`. Це ті ж самі тести, але написані з використанням `rspec_api_documentation` DSL, таким чином gem може генерувати документацію на основі тестів.

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

Додайте, будь ласка, конфігурацію для гему в файл `spec/spec_helper.rb`

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

Тепер ви можете видалити файл `spec/service_spec.rb` і використовувати замість нього файл `spec/acceptance/service_spec.rb`.

Створіть файл `Rakefile` в теці `users`.

```ruby
require_relative 'service'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Нам не потрібні задачі для створення документації в `production` середовищі, тому ми не завантажуємо їх у Rakefile в цьому випадку.

Ви можете побачити всі доступні `rake` задачі за допомогою команди `rake -T`

    $ rake -T

    rake docs:generate          # Generate API request documentation from API specs
    rake docs:generate:ordered  # Generate API request documentation from API specs (ordered)

І ми можемо згенерувати документацію за допомогою будь-якої з цих двох задач

    $ rake docs:generate

Документація в форматі HTML буде збережена в теці `doc`. Вам варто використовувати імена для тестів більш підходящі для людей (програмістів, які будуть користуватися вашим сервісом).

## <a name="exclude-specific-test-examples-from-documentation"></a>Виключення деяких тестів з документації

Ви можете виключити певні тестові приклади з документації за допомогою опції `document: false`, наприклад:

```ruby
# This example does not fall into the documentation.
example_request "no token - no user", document: false do
  expect(status).to eq 403
  expect(response_body).to eq '{"message":"Access Forbidden"}'
end
```

## <a name="summary"></a>Резюме

Ми використовували gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для створення автоматично згенерованої документації на основі RSpec тестів. Іноді це може бути корисним.
