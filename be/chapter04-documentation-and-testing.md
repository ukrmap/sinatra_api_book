Кіраўнік #4. Дакументацыя і тэставанне
======================================

Калі вы ствараеце API вам таксама неабходна падаць дакументацыю (спецыфікацыі) для яго. І вам таксама неабходна ствараць тэсты, якія з'яўляюцца гарантыяй надзейнасці, таксама як і спецыфікацыяй. У гэтай частцы мы пастараемся аб'яднаць дакументацыю і тэсты.

Мы створым мінімалісцкім ўнутры API (users service), якое мы будзем выкарыстоўваць у іншых серфисах для аўтэнтыфікацыі.

## <a name="planning-interface"></a>Планаванне інтэрфейсу

Users API будзе мець адзін URI-шлях, які будзе вяртаць JSON-прадстаўленне канкрэтнага карыстальніка. Карыстальнік (можа быць іншай вэб-сэрвіс) нашага API павінен перадаць правільны ключ ідэнтыфікацыі ў HTTP загалоўку запыту. Ключ (token) праверкі сапраўднасці - гэта некаторая паслядоўнасць знакаў - радок, якая вядомая, толькі карыстачу і вэб-сэрвісу. У рэальнасці, калі ключ скончыўся, карыстальнік павінен запытаць новы, але мы не будзем создаавать гэты фунцционал і будзем выкарыстоўваць вечныя ключы для прастаты.

Мы будзем выкарыстоўваць карыстальнікаў двух тыпаў: "AdminUser" і "RegularUser" (на самай справе толькі адзін "AdminUser" і адзін "RegularUser").

Прыклад запыту да сэрвісу:

    $ curl -X GET "localhost:4567/api/v1/users/me.json" -H "Authorization: OAuth user_token"

    {"user":{"type":"RegularUser"}}

Дзе радок `user_token` павінна быць заменена актуальным токенаў карыстальніка. Токен - гэта значэнне HTTP загалоўка "Authorization" з прэфіксам "OAuth" (для [OAuth](https://ru.wikipedia.org/wiki/OAuth) або [OAuth 2.0](http://oauth.net/2/ )).
Калі токен не з'яўляецца правільным сэрвіс павінен вярнуць HTTP адказ c кодам 401 (Unauthorized) і адпаведным паведамленнем пра памылку ў фармаце JSON. Калі токен не перададзены сэрвіс павінен вярнуць HTTP адказ з кодам 403 (Forbidden) і таксама адпаведным паведамленнем пра памылку ў фармаце JSON.

## <a name="create-service"></a>Стварэнне сэрвісу

Стварыце, калі ласка, тэчку `users` дзесьці ў сістэме, гэта будзе каранёвая тэчка для новага сэрвісу. Стварыце ў ёй файл `Gemfile` са наступным змесцівам:

```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'rspec'
gem 'rack-test'
gem 'rspec_api_documentation'
```

Зайдзіце ў тэчку `users` ў тэрмінале і запусціўшы каманду:

    $ bundle install

Стварыце файл `service.rb` ў тэчцы` users` з наступным змесцівам:

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

Мы дадалі адзін URL-маршрут (з блокам). Блок змяшчае значэнне двух токенаў (якія, відавочна, не скончацца) непасрэдна ў кодзе. Ён вяртае карыстальніка тыпу "AdminUser" ці "RegularUser" у залежнасці ад токена. Мы можам запусціць сэрвіс з дапамогай каманды `ruby service.rb` і пратэставаць яго (з іншай ўкладкі або вокны тэрмінала):

    $ curl -X GET "localhost:4567/api/v1/users/me.json" \
      -H "Authorization: OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86"

    {"user":{"type":"RegularUser"}}

Мы падзялілі `curl` запыт на два радкі сімвалам "\" (зваротная косая рыса).

Мы таксама можам праверыць адказ сэрвісу, калі перададзены няправільны токен або токен не перададзены. Выкарыстоўвайце `-i` сцяг, каб убачыць HTTP адказ з загалоўкамі.

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

## <a name="create-tests"></a>Стварэнне тэстаў

Стварыце, калі ласка, тэчку `spec` ў тэчцы `users` і файл `service_spec.rb` ў тэчцы `spec` са наступным змесцівам:

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

І файл `spec_helper.rb` (таксама ў тэчцы `spec`)

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

Цяпер мы можам запусціць нашы тэсты:

    $ rspec
    ....

    Finished in 0.06302 seconds (files took 0.3102 seconds to load)
    4 examples, 0 failures

## <a name="create-documentation-from-tests"></a>Стварэнне дакументацыі на аснове тэстаў

Тэст правярае усё тое, пра што мы гаварылі напачатку часткі. Так што, калі вы ў стане добра пісаць тэсты-спецыфікацыі, то можна лічыць добрай ідэяй стварэнне дакументацыі з іх. Мы будзем выкарыстоўваць gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для гэтай мэты (gem быў ужо дададзены ў `Gemfile`).

Мы павінны перапісаць тэсты ў меркаванай форме з выкарыстаннем `rspec_api_documentation` DSL і змясціць іх у адпаведную тэчку - `spec/acceptance`. Мы павінны наладзіць `rspec_api_documentation`. І мы павінны стварыць `rake` задачы для стварэння дакументацыі. Давайце зробім гэта.

Стварыце, калі ласка, тэчку `rspec/acceptance` і дадайце ў яе файл `service_spec.rb`. Гэта тыя ж самыя тэсты, але напісанне з выкарыстаннем `rspec_api_documentation` DSL, такім чынам gem можа генераваць дакументацыю на падставе тэстаў.

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

Дадайце, калі ласка, канфігурацыю для гема ў файл `spec/spec_helper.rb`

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

Цяпер вы можаце выдаліць файл `spec/service_spec.rb` і выкарыстоўваць замест яго файл `spec/acceptance/service_spec.rb`.

Стварыце файл `Rakefile` ў тэчцы` users`.

```ruby
require_relative 'service'

unless ENV['RACK_ENV'].to_s == 'production'
  require 'rspec_api_documentation'
  load 'tasks/docs.rake'
end
```

Нам не патрэбныя задачы для стварэння дакументацыі ў `production` асяроддзі, таму мы не загружаем іх у Rakefile ў гэтым выпадку.

Вы можаце ўбачыць усе даступныя `rake` задачы з дапамогай каманды `rake -T`

    $ rake -T

    rake docs:generate          # Generate API request documentation from API specs
    rake docs:generate:ordered  # Generate API request documentation from API specs (ordered)

І мы можам згенераваць дакументацыю з дапамогай любой з гэтых двух задач

    $ rake docs:generate

Дакументацыя ў фармаце HTML будзе захоўвацца ў тэчцы `doc`. Вам варта выкарыстаць імёны для тэстаў больш прыдатныя для людзей (праграмістаў, якія будуць карыстацца вашым сэрвісам).

## <a name="exclude-specific-test-examples-from-documentation"></a>Выключэнне некаторых тэстаў з дакументацыі

Вы можаце выключыць пэўныя тэставыя прыклады з дакументацыі з дапамогай опцыі `document: false`, напрыклад:

```ruby
# This example does not fall into the documentation.
example_request "no token - no user", document: false do
  expect(status).to eq 403
  expect(response_body).to eq '{"message":"Access Forbidden"}'
end
```

## <a name="summary"></a>Резюме

Мы выкарыстоўвалі gem [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation) для стварэння аўтаматычна згенераванай дакументацыі на аснове RSpec тэстаў. Часам гэта можа быць карысным.
