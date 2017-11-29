Глава #9. Взаимодействия и Клиенты веб-сервисов
===============================================

Сервис Zip-кодов почти готов к использованию реальными клиентами. Он может быть использован любым приложением, которое может выполнять HTTP запросы. Это приложение, таким образом, становится клиентом веб-сервиса. JavaScript приложения, работающие в веб-браузере естественно вписывается в эту архитектуру. Приложение ruby также может действовать как клиент веб-сервиса. Любой веб-сервис может изнутри использовать другой веб-сервис. Сервис Zip-кодов уже использует users сервис для автроризации.

В этой главе мы создадим клиент для веб сервиса Zip-кодов на ruby. Но сначала мы сделаем небольшие изменения в самом сервисе.

## <a name="rest-conventions"></a>Конвенции REST

Мы создали сервис Zip-кодов следуя конвенциям REST, мы используем методы HTTP протокола, чтобы определить метод CRUD и используем URL, чтобы определить ресурс, над которым следует выполнить метод. Но мы используем разные адреса для обновления и получения ресурса - обновление Zip-кода происходит по ID (путь `"/api/v1/zip_codes/:id.json"`), получение - по атрибуту `zip` (путь `"/api/v1/zip_codes/:zip.json"`). Это было сделано в обучающих целях (или по небрежности автора), и теперь мы должны следовать соглашениям. Мы изменим путь для получения Zip-кода в файле `app/controllers/zip_codes_controller.rb`.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  authorize! :create, zip_code
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes.json" do
  zip_codes = ZipCode.search(params.extract!('zip_start', 'street_name_eq',
    'building_number_eq', 'city_eq', 'state_eq')).result.limit(20)
  zip_codes.to_json
end

# Route was changed to the same like in Update and Delete actions
get "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  zip_code = ZipCode.find(params[:id])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  param :zip_code, Hash, required: true
  load_and_authorize! ZipCode
  @zip_code.update_attributes!(params[:zip_code]) if params[:zip_code].any?
  @zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  load_and_authorize! ZipCode
  @zip_code.destroy!
end
```

Также необходимо исправить тесты получения Zip-кода (GET) в файле `spec/acceptance/zip_codes_spec.rb`

```ruby
require "spec_helper"

resource 'ZipCode' do
  
  # tests for create (post) action

  get "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    context "Public User" do
      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end

      example "Read Zip Code that does not exist", document: nil do
        do_request(id: 8889)

        expect(status).to eq 404
        expect(response_body).to eq '{"message":"Record not found"}'
      end

      example "Read Zip Code provide invalid format zip id", document: nil do
        do_request(id: 's1234')
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 400
        expect(json_response[:message]).to eq 'Invalid Parameter: id'
        expect(json_response[:errors][:id]).to eq "'s1234' is not a valid Integer"
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")', document: nil do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end
    end

    context "Admin User", document: nil do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"AdminUser"}}') }

      example "Read Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end
    end
  end

  # tests for update (put) and delete actions

end
```

## <a name="curl"></a>Curl

Давайте быстро проверим с помощью утилиты `curl`, что сервис работает, как ожидалось. `Curl` также есть клиентом веб-сервиса. Пожалуйста, запустите users сервис на порту 4545 (выполнить `ruby service.rb -p 4545` в терминале из директории users) и сервис Zip-кодов на порту 4567 (выполнить `ruby application.rb` из директории `zip_codes`). Мы готовы к проверке.

Получение списка Zip-кодов

    $ curl "http://localhost:4567/api/v1/zip_codes.json" \
      -X GET

    [{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming","created_at":"2015-02-15T09:02:25.383Z","updated_at":"2015-02-15T09:02:25.383Z"}},{"zip_code":{"id":403,"zip":"98189-4795","street_name":"Lucie Falls"...

Получение одного Zip-кода по ID

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X GET

    {"zip_code":{"id":401,"zip":"63109","street_name":"Candido Loop","building_number":"897","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T10:48:59.680Z"}}

Изменение нескольких атрибутов Zip-кода (`street_name` и `building_number`)

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X PUT \
      -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
      -H "Content-Type: application/json" \
      -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T13:19:52.762Z"}}

Все работает, как и ожидалось, давайте создадим `ruby` клиент для сервиса.

Добавьте следующую строку в файл `zip_codes/application.rb` если вы также хотите проверить обработку ошибок в режиме `development`.

```ruby
disable :show_exceptions # in production it is false, so you probably do not need it
```

## <a name="web-service-client"></a>Клиент веб-сервиса

Мы создадим клиент в сессии `irb` используя gem `activeresource`. Установите gem `activeresource`

    $ gem install activeresource

Начните `irb` сессию

    $ irb

```ruby
require "active_resource"

class ZipCode < ActiveResource::Base
  self.format = :json
  self.include_root_in_json = true 
  self.site = "http://localhost:4567"
  self.prefix = "/api/v1/"
end
```

Это все, что нужно - работа сделана, теперь мы можем все проверить в той же сессии `irb`.

Получить один Zip-код по адресу `http://localhost:4567/api/v1/zip_codes/401.json`

```ruby
zip_code = ZipCode.find(401)
zip_code.zip # => "63109" 
zip_code.street_name # => "Candido Loop" 
zip_code.building_number # => "897" 
zip_code.city # => "New Hoyt" 
zip_code.state # => "Utah"
```

Получить список Zip-кодов по адресу `http://localhost:4567/api/v1/zip_codes.json`

```ruby
zip_codes = ZipCode.find(:all)
zip_codes = ZipCode.find(:all, params: { state_eq: "Massachusetts" })
zip_codes = ZipCode.find(:all, params: { zip_start: "119" })
```

Запросить список (тот же URL), но вернуть только одну запись

```ruby
zip_codes = ZipCode.find(:first, params: { zip_start: "119" })
```

Редактирование записи Zip-кода (администратором)

```ruby
ZipCode.headers["Authorization"] = "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
zip_code.city = "New Johnathan"
zip_code.save
```

Обратите внимание на HTTP заголовок "Authorization".

Удаление Zip-кода

```ruby
zip_code = ZipCode.find(101186)
zip_code.destroy
```

Создание Zip-кода

```ruby
zip_code = ZipCode.create(zip: "82470-2132", street_name: "Micheal Street",
  building_number: "911", city: "South Madalyn", state: "Louisiana")

zip_code.id # => 101187 
```

## <a name="cors"></a>Поддержка CORS

Если приложение работающее в веб-браузере должно иметь доступ к веб-сервису, мы должны позволить Cross-Origin Resource Sharing (CORS). В противном случае, только приложения, который запущены на том же URL, что и веб-сервис смогут получить к нему доступ.

Мы можем использовать `Rack Middleware` для этого.

Добавьте гем `rack-cors` в `Gemfile` и выполните `bundle install`.

```ruby
gem 'rack-cors', :require => 'rack/cors'
```

Добавьте следующий код в файл `application.rb`

```ruby
use Rack::Cors do
  allow do
    origins '*'
    resource '/*', headers: :any, methods: [:post, :put, :get, :delete, :options]
  end
end
```

После этого сайт с любым URL сможет иметь доступ к веб-сервису.

## <a name="summary"></a>Резюме

Мы создали клиент для сервиса Zip-кодов с помощью [activeresource](https://github.com/rails/activeresource).
