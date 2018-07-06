Кіраўнік #9. Ўзаемадзеяння і Кліенты вэб-сэрвісаў
=================================================

Сэрвіс Zip-кодаў амаль гатовы да выкарыстання рэальнымі кліентамі. Ён можа быць выкарыстаны любым прыкладаннем, якое можа выконваць HTTP запыты. Гэта дадатак, такім чынам, становіцца кліентам вэб-сэрвісу. JavaScript прыкладання, якія працуюць у вэб-браўзэры натуральна ўпісваецца ў гэтую архітэктуру. Дадатак ruby ​​таксама можа дзейнічаць як кліент вэб-сэрвісу. Любы вэб-сэрвіс можа знутры выкарыстаць іншы вэб-сэрвіс. Сэрвіс Zip-кодаў ўжо выкарыстоўвае users сэрвіс для автроризации.

У гэтай частцы мы створым кліент для вэб сэрвісу Zip-кодаў на ruby. Але спачатку мы зробім невялікія змены ў самім сэрвісе.

## <a name="rest-conventions"></a>Канвенцыі REST

Мы стварылі сэрвіс Zip-кодаў вынікаючы канвенцый REST, мы выкарыстоўваем метады HTTP пратаколу, каб вызначыць метад CRUD і выкарыстоўваем URL, каб вызначыць рэсурс, над якім варта выканаць метад. Але мы выкарыстоўваем розныя адрасы для абнаўлення і атрымання рэсурсу - абнаўленне Zip-коду адбываецца па ID (шлях `"/api/v1/zip_codes/:id.json"`), атрыманне - па атрыбуту `zip` (шлях `"/api/v1/zip_codes/:zip.json"`). Гэта было зроблена ў навучальных мэтах (або па нядбайнасці аўтара), і зараз мы павінны прытрымлівацца пагадненням. Мы зменім шлях для атрымання Zip-кода ў файле `app/controllers/zip_codes_controller.rb`.

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

Таксама неабходна выправіць тэсты атрымання Zip-кода (GET) у файле `spec/acceptance/zip_codes_spec.rb`

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

      example "Read Zip Code that does not exist", document: false do
        do_request(id: 8889)

        expect(status).to eq 404
        expect(response_body).to eq '{"message":"Record not found"}'
      end

      example "Read Zip Code provide invalid format zip id", document: false do
        do_request(id: 's1234')
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 400
        expect(json_response[:message]).to eq 'Invalid Parameter: id'
        expect(json_response[:errors][:id]).to eq "'s1234' is not a valid Integer"
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")', document: false do
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

    context "Admin User", document: false do
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

Давайце хутка праверым з дапамогай утыліты `curl`, што сэрвіс працуе, як чакалася. `Curl` таксама ёсць кліентам вэб-сэрвісу. Калі ласка, запусціце users сэрвіс на порце 4545 (выканаць `ruby service.rb -p 4545` ў тэрмінале з дырэкторыі users) і сэрвіс Zip-кодаў на порце 4567 (выканаць` ruby application.rb` з дырэкторыі `zip_codes`). Мы гатовыя да праверкі.

Атрыманне спісу Zip-кодаў

    $ curl "http://localhost:4567/api/v1/zip_codes.json" \
      -X GET

    [{"zip_code":{"id":402,"zip":"40664-8387","street_name":"Wuckert Mall","building_number":"2294","city":"New Aiyanatown","state":"Wyoming","created_at":"2015-02-15T09:02:25.383Z","updated_at":"2015-02-15T09:02:25.383Z"}},{"zip_code":{"id":403,"zip":"98189-4795","street_name":"Lucie Falls"...

Атрыманне аднаго Zip-кода па ID

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X GET

    {"zip_code":{"id":401,"zip":"63109","street_name":"Candido Loop","building_number":"897","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T10:48:59.680Z"}}

Змена некалькіх атрыбутаў Zip-кода (`street_name` і `building_number`)

    $ curl "http://localhost:4567/api/v1/zip_codes/401.json" \
      -X PUT \
      -H "Authorization: OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf" \
      -H "Content-Type: application/json" \
      -d '{"zip_code":{"street_name":"Wuckert Mall","building_number":"2294"}}'

    {"zip_code":{"id":401,"zip":"63109","street_name":"Wuckert Mall","building_number":"2294","city":"New Hoyt","state":"Utah","created_at":"2015-02-15T09:02:25.374Z","updated_at":"2015-02-20T13:19:52.762Z"}}

Усё працуе, як і чакалася, давайце створым `ruby` кліент для сэрвісу.

Дадайце наступны радок у файл `zip_codes/application.rb` калі вы таксама хочаце праверыць апрацоўку памылак у рэжыме `development`.

```ruby
disable :show_exceptions # in production it is false, so you probably do not need it
```

## <a name="web-service-client"></a>Кліент вэб-сэрвісу

Мы створым кліент у сесіі `irb` выкарыстоўваючы gem `activeresource`. Усталюйце gem `activeresource`

    $ gem install activeresource

Пачніце `irb` сесію

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

Гэта ўсё, што трэба - праца зробленая, зараз мы можам усё праверыць у той жа сесіі `irb`.

Атрымаць адзін Zip-код па адрасе `http://localhost:4567/api/v1/zip_codes/401.json`

```ruby
zip_code = ZipCode.find(401)
zip_code.zip # => "63109"
zip_code.street_name # => "Candido Loop"
zip_code.building_number # => "897"
zip_code.city # => "New Hoyt"
zip_code.state # => "Utah"
```

Атрымаць спіс Zip-кодаў па адрасе `http://localhost:4567/api/v1/zip_codes.json`

```ruby
zip_codes = ZipCode.find(:all)
zip_codes = ZipCode.find(:all, params: { state_eq: "Massachusetts" })
zip_codes = ZipCode.find(:all, params: { zip_start: "119" })
```

Запытаць спіс (той жа URL), але вярнуць толькі адзін запіс

```ruby
zip_codes = ZipCode.find(:first, params: { zip_start: "119" })
```

Рэдагаванне запісу Zip-кода (адміністратарам)

```ruby
ZipCode.headers["Authorization"] = "OAuth 562f9fdef2c4384e4e8d59e3a1bcb74fa0cff11a75fb9f130c9f7a146a003dcf"
zip_code.city = "New Johnathan"
zip_code.save
```

Звярніце ўвагу на HTTP загаловак "Authorization".

Выдаленне Zip-кода

```ruby
zip_code = ZipCode.find(101186)
zip_code.destroy
```

Стварэнне Zip-кода

```ruby
zip_code = ZipCode.create(zip: "82470-2132", street_name: "Micheal Street",
  building_number: "911", city: "South Madalyn", state: "Louisiana")

zip_code.id # => 101187
```

## <a name="cors"></a>Падтрымка CORS

Калі дадатак якое працуе ў вэб-браўзэры павінна мець доступ да вэб-сэрвісу, мы павінны дазволіць Cross-Origin Resource Sharing (CORS). У адваротным выпадку, толькі прыкладання, які запушчаны на тым жа URL, што і вэб-сэрвіс змогуць атрымаць да яго доступ.

Мы можам выкарыстоўваць `Rack Middleware` для гэтага.

Дадайце гем `rack-cors` ў `Gemfile` і выканайце `bundle install`.

```ruby
gem 'rack-cors', :require => 'rack/cors'
```

Дадайце наступны код у файл `application.rb`

```ruby
use Rack::Cors do
  allow do
    origins '*'
    resource '/*', headers: :any, methods: [:post, :put, :get, :delete, :options]
  end
end
```

Пасля гэтага сайт з любым URL зможа мець доступ да вэб-сэрвісу.

## <a name="summary"></a>Рэзюмэ

Мы стварылі кліент для сэрвісу Zip-кодаў з дапамогай [activeresource](https://github.com/rails/activeresource).
