Глава # 9. Взаємодії та Клієнти веб-сервісів
============================================

Сервіс Zip-кодів майже готовий до використання реальними клієнтами. Він може бути використаний будь-якою програмою, яка може виконувати HTTP запити. Ця програма, таким чином, стає клієнтом веб-сервісу. JavaScript програми, що працюють у веб-браузері природно вписуються в цю архітектуру. програма на ruby також може діяти як клієнт веб-сервісу. Будь-який веб-сервіс може зсередини використовувати інший веб-сервіс. Сервіс Zip-кодів вже використовує users сервіс для автрорізаціі.

У цій главі ми створимо клієнт для веб сервісу Zip-кодів на ruby. Але спочатку ми зробимо невеликі зміни в самому сервісі.

## <a name="rest-conventions"></a>Конвенції REST

Ми створили сервіс Zip-кодів слідуючи конвенціям REST, ми використовуємо методи HTTP протоколу, щоб визначити метод CRUD і використовуємо URL, щоб визначити ресурс, над яким слід виконати метод. Але ми використовуємо різні адреси для оновлення та отримання ресурсу - оновлення Zip-коду відбувається по ID (шлях `"/api/v1/zip_codes/:id.json"`), отримання - по атрибуту `zip` (шлях `"/api/v1/zip_codes/:zip.json"`). Це було зроблено в навчальних цілях (або по недбалості автора), і тепер ми повинні слідувати конвенціям. Ми змінимо шлях для отримання Zip-коду у файлі `app/controllers/zip_codes_controller.rb`.

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

Також необхідно виправити тести отримання Zip-коду (GET) у файлі `spec/acceptance/zip_codes_spec.rb`

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

Давайте швидко перевіримо за допомогою утиліти `curl`, що сервіс працює, як і очікується. `Curl` також є клієнтом веб-сервісу. Будь ласка, запустіть users сервіс на порту 4545 (виконати `ruby service.rb -p 4545` в терміналі з директорії users) і сервіс Zip-кодів на порту 4567 (виконати `ruby application.rb` з директорії `zip_codes`). Ми готові до перевірки.

Отримання списку Zip-кодів

    $ curl "http://localhost:4567/api/v1/zip_codes.json" \
      -X GET

    [{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming","created_at":"2015-02-15T09:02:25.383Z","updated_at":"2015-02-15T09:02:25.383Z"}},{"zip_code":{"id":403,"zip":"98189-4795","street_name":"Lucie Falls"...

Отримання одного Zip-коду по ID

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X GET

    {"zip_code":{"id":401,"zip":"63109","street_name":"Candido Loop","building_number":"897","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T10:48:59.680Z"}}

Зміна декількох атрибутів Zip-коду (`street_name` та `building_number`)

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X PUT \
      -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
      -H "Content-Type: application/json" \
      -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T13:19:52.762Z"}}

Все працює, як і очікувалося, давайте створимо `ruby` клієнт для сервісу.

Додайте наступний рядок у файл `zip_codes/application.rb` якщо ви також хочете перевірити обробку помилок в режимі `development`.

```ruby
disable :show_exceptions # in production it is false, so you probably do not need it
```

## <a name="web-service-client"></a>Клієнт веб-сервісу

Ми створимо клієнт у сесії `irb` використовуючи gem  activeresource`. Встановіть gem `activeresource`

    $ gem install activeresource

Почніть `irb` сесію

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

Це все, що потрібно було зробити, тепер ми можемо все перевірити в тій самій сесії `irb`.

Отримати один Zip-код за адресою `http://localhost:4567/api/v1/zip_codes/401.json`

```ruby
zip_code = ZipCode.find(401)
zip_code.zip # => "63109" 
zip_code.street_name # => "Candido Loop" 
zip_code.building_number # => "897" 
zip_code.city # => "New Hoyt" 
zip_code.state # => "Utah"
```

Отримати список Zip-кодів за адресою `http://localhost:4567/api/v1/zip_codes.json`

```ruby
zip_codes = ZipCode.find(:all)
zip_codes = ZipCode.find(:all, params: { state_eq: "Massachusetts" })
zip_codes = ZipCode.find(:all, params: { zip_start: "119" })
```

Запросити список (той же URL), але повернути тільки один запис

```ruby
zip_codes = ZipCode.find(:first, params: { zip_start: "119" })
```

Редагування запису Zip-коду (адміністратором)

```ruby
ZipCode.headers["Authorization"] = "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
zip_code.city = "New Johnathan"
zip_code.save
```

Зверніть увагу на HTTP заголовок "Authorization".

Видалення Zip-коду

```ruby
zip_code = ZipCode.find(101186)
zip_code.destroy
```

Створення Zip-коду

```ruby
zip_code = ZipCode.create(zip: "82470-2132", street_name: "Micheal Street",
  building_number: "911", city: "South Madalyn", state: "Louisiana")

zip_code.id # => 101187 
```

## <a name="cors"></a>Підтримка CORS

Якщо додаток працює у веб-браузері повинна мати доступ до веб-сервісу, ми повинні дозволити Cross-Origin Resource Sharing (CORS). В іншому випадку, тільки додатки, який запущені на тому ж URL, що і веб-сервіс зможуть отримати до нього доступ.

Ми можемо використовувати `Rack Middleware` для цього.

Додайте гем `rack-cors` в `Gemfile` і виконайте `bundle install`.

```ruby
gem 'rack-cors', :require => 'rack/cors'
```

Додайте наступний код в файл `application.rb`

```ruby
use Rack::Cors do
  allow do
    origins '*'
    resource '/*', headers: :any, methods: [:post, :put, :get, :delete, :options]
  end
end
```

Після цього сайт з будь-яким URL зможе мати доступ до веб-сервісу.

## <a name="summary"></a>Резюме

Ми створили клієнт для сервісу Zip-кодів за допомогою [activeresource](https://github.com/rails/activeresource).
