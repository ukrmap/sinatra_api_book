Кіраўнік #6. Валідацыю і коды стану HTTP
========================================

Обробка некарэктныя увядзеннем даних є важливим аспектам функцыянавання вэб-сервісу, а також повідомлення пра помилки для клієнта. [Коды стану HTTP](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) - гэта агульная простая канцэпцыя для паведамлення кліента аб статусе апрацоўкі HTTP запыту. Паведамленне пра памылку ў фармаце JSON дапаўняе HTTP-адказ і ўдакладняе памылку для канчатковых карыстальнікаў.

Вось спіс кодаў стану HTTP, якія мы будзем выкарыстоўваць:

* 200, 201 - паспяхова: "добра" - OK і «створана» (201 - для `POST` запытаў)
* 404 - «не знойдзена»
* 400, 422 - «дрэнны, нягодны запыт», «необрабатываемые асобнік»
* 401, 403 - «несанкцыянаванага», «забаронена»
* 500 - «унутраная памылка сервера»

400 і 422 вельмі падобныя, мы будзем выкарыстоўваць 400, каб паведаміць аб няслушных параметрах запыту ў цэлым (няправільны тып ці значэнне па-за дыяпазону) і 422 для памылак валідацыю.

401 і 403 таксама падобныя. Мы будзем выкарыстоўваць 401, калі карыстальнік адпраўляе няправільны або са скончаным тэрмінам дзеяння токен для аўтэнтыфікацыі. І 403, каб паведаміць карыстальніка (які магчыма увайшоў у сістэму) пра памылку аўтарызацыі (забарона на выкананне аперацыі).

## <a name="500-internal-server-error"></a>500 - Унутраная памылка сервера

Пры бягучай рэалізацыі вэб-сэрвісу, калі адбываецца раптоўная памылка, сэрвіс вяртае 500-ю памылку без цела адказу. Распрацоўваючы JSON-сэрвіс, мы павінны прадугледзець апавяшчэнне для кліента з адпаведным паведамленнем пра памылку ў фармаце JSON (у файле `application.rb`).

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="404-not-found"></a>404 - Не знойдзена

Калі кліент дае ідэнтыфікатар (ID) Zip-кода, які не існуе ў базе дадзеных, ActiveRecord метад `find` вяртае памылку` ActiveRecord::RecordNotFound` і вэб-сэрвіс вяртае 500-й код стану HTTP. Гэта не добра. Сэрвіс павінен вяртаць 404-й код статусу і адпаведнае паведамленне пра памылку.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Catch properly ActiveRecord::RecordNotFound
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="422-unprocessable-entity"></a>422 - Необрабатываемые асобнік

Калі мы паспрабуем захаваць мадэль з няправільнымі данымі (напрыклад, няправільны фармат ZIP) ActiveRecord верне памылку `ActiveRecord::RecordInvalid`. У гэтым выпадку сэрвіс павінен вяртаць 422-ой код статусу HTTP і паведамленне аб спецыфічных памылках валідацыю.

```ruby
require 'bundler/setup'
Bundler.require :default, (ENV['RACK_ENV'] || :development).to_sym
puts "Loaded #{Sinatra::Application.environment} environment"

set :root, File.dirname(__FILE__)
use Rack::CommonLogger, File.new(File.join(settings.root, 'log',
  "#{settings.environment}.log"), 'a+').tap { |f| f.sync = true }

Dir[File.join(settings.root, "app/{models,controllers}/*.rb")].each { |f| require f }

use Rack::PostBodyContentTypeParser
before { content_type :json }
ActiveRecord::Base.include_root_in_json = true

# Catch properly ActiveRecord::RecordNotFound
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }

# Catch validation errors
error(ActiveRecord::RecordInvalid) do
  [422, { message: "Validation errors occurred",
          errors:  env['sinatra.error'].record.errors.messages }.to_json ]
end

# Respond with error message at unexpected exception (HTTP 500 status code will be added by sinatra).
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

## <a name="tests"></a>Тэсты

Мы можам праверыць апрацоўку памылак у `acceptance` тэстах. Файл `spec/acceptance/zip_codes_spec.rb`

```ruby
require "spec_helper"

resource 'ZipCode' do
  post "/api/v1/zip_codes.json" do
    header "Content-Type", "application/json"

    parameter :zip, "Zip", scope: :zip_code, required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    # let(:valid_attributes) do
    #   { zip: "35761-7714", street_name: "Lavada Creek",
    #       building_number: "88871", city: "New Herminaton", state: "Rhode Island" }
    # end
    let(:valid_attributes) { attributes_for(:zip_code) }
    let(:new_zip_code) { ZipCode.last }

    example "Create Zip Code" do
      do_request(zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 201
      expect(json_response[:zip_code].values_at(*valid_attributes.keys)).to eq valid_attributes.values
      expect(new_zip_code).to be_present
      expect(new_zip_code.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end

    example "Create Zip Code with invalid params", document: false do
      do_request(zip_code: { zip: "1234" })

      expect(status).to eq 422
      expect(response_body).to eq '{"message":"Validation errors occurred","errors":{"zip":["is invalid"]}}'
      expect(new_zip_code).to be_nil
    end
  end

  get "/api/v1/zip_codes/:zip.json" do
    parameter :zip, "Zip", scope: :zip_code, required: true

    let(:zip_code) { create(:zip_code) }

    example "Read Zip Code" do
      do_request(zip: zip_code.zip)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
        zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
    end

    example "Read Zip Code that does not exist", document: false do
      do_request(zip: '12345-6789')

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end
  end

  put "/api/v1/zip_codes/:id.json" do
    header "Content-Type", "application/json"

    parameter :id, "Record ID", required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    let(:zip_code) { create(:zip_code) }
    let(:valid_attributes) { attributes_for(:zip_code) }

    example "Update Zip Code" do
      do_request(id: zip_code.id, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:zip, :street_name, :building_number, :city, :state)).to eq(
        valid_attributes.values_at(:zip, :street_name, :building_number, :city, :state))
      expect(zip_code.reload.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end

    example "Update Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end
  end

  delete "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    example "Delete Zip Code" do
      do_request(id: zip_code.id)

      expect(status).to eq 200
      expect(ZipCode.where(id: zip_code.id)).to be_empty
    end

    example "Delete Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end
  end
end
```

## <a name="400-bad-request"></a>400 - Няправільны запыт

Часам нам трэба забяспечыць правільнасць тыпаў параметраў, якія перадае кліент. Мы можам прадухіліць ўзаемадзеянне з базай дадзеных пры некарэктных ўваходных дадзеных і неадкладна вярнуць паведамленне пра памылку кліенту. Мы будзем выкарыстоўваць гем [sinatra-param](https://github.com/mattt/sinatra-param) для праверкі і пераўтварэнні тыпаў параметраў у `sinatra`.

Калі ласка, дадайце радок `gem 'sinatra-param'` ў` Gemfile`

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
gem 'rack-contrib', git: 'https://github.com/rack/rack-contrib'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'
# Parameter Validation & Type Coercion for Sinatra
gem 'sinatra-param'

group :development, :test do
  gem 'thin'
  gem 'pry-debugger'
  gem 'rspec_api_documentation'
end

group :test do
  gem 'rspec'
  gem 'shoulda'
  gem 'factory_girl'
  gem 'database_cleaner'
  gem 'rack-test'
  gem 'faker'
end
```

І выканайце `bundle install`

Калі кліент прадастаўляе параметр `zip_code` няправільнага тыпу (не `Hash`) вэб-сэрвіс вяртае памылку 500 (напрыклад, ``NoMethodError: undefined method `stringify_keys 'for" STRING ": String`` для тыпу `String`). Мы можам прадухіліць гэтыя паводзіны з дапамогай наступнага дэкларатыўнага кода ў файле `app/controllers/zip_codes_controller.rb`.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true # ensure params[:zip_code] is a Hash
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :zip_code, Hash, required: true # ensure params[:zip_code] is a Hash
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

Мы можам пайсці далей і правяраць рэгулярным выразам параметр `zip` ў `get` роуте. Гэта прадухіляе зварот у базу, калі параметр мае няправільны фармат. І вэб-сэрвіс вяртае код HTTP статусу 400 замест 404, а таксама больш падыходнае паведамленне пра памылку (`"Параметр павінен адпавядаць фармату"` замест `"Запіс не знойдзена"`, якое прадугледжвае, што запіс магчыма была выдаленая).

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  param :zip, String, format: /\A\d{5}(?:-\d{4})?\Z/ # route logic stops here if zip has wrong format
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

У мяне ёсць абмежаванні для цэлага тыпу ў базе дадзеных. Калі кліент дае ідэнтыфікатар больш чым `2147483647` (што ў двайковай сістэме роўна` 111111111111111111111111111111`) `activerecord` вяртае памылку` PostgreSQL` і вэб-сэрвіс адказвае з 500-й HTTP памылкай. Можна пазбегнуць гэтага з наступнай фільтраваннем параметраў.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  zip_code.save!
  status 201
  zip_code.to_json
end

get "/api/v1/zip_codes/:zip.json" do
  param :zip, String, format: /\A\d{5}(?:-\d{4})?\Z/
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647 # 0b111111111111111111111111111111
  param :zip_code, Hash, required: true
  zip_code = ZipCode.find(params[:id])
  zip_code.update_attributes!(params[:zip_code])
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647 # 0b111111111111111111111111111111
  zip_code = ZipCode.find(params[:id])
  zip_code.destroy!
end
```

Абноўленыя тэсты (файл `spec/acceptance/zip_codes_spec.rb`)

```ruby
require "spec_helper"

resource 'ZipCode' do
  post "/api/v1/zip_codes.json" do
    header "Content-Type", "application/json"

    parameter :zip, "Zip", scope: :zip_code, required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    # let(:valid_attributes) do
    #   { zip: "35761-7714", street_name: "Lavada Creek",
    #       building_number: "88871", city: "New Herminaton", state: "Rhode Island" }
    # end
    let(:valid_attributes) { attributes_for(:zip_code) }
    let(:new_zip_code) { ZipCode.last }

    example "Create Zip Code" do
      do_request(zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 201
      expect(json_response[:zip_code].values_at(*valid_attributes.keys)).to eq valid_attributes.values
      expect(new_zip_code).to be_present
      expect(new_zip_code.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end

    example "Create Zip Code with invalid params", document: false do
      do_request(zip_code: { zip: "1234" })

      expect(status).to eq 422
      expect(response_body).to eq '{"message":"Validation errors occurred","errors":{"zip":["is invalid"]}}'
      expect(new_zip_code).to be_nil
    end

    example "Create Zip Code provide not Hash zip_code params", document: false do
      do_request(zip_code: "STRING")

      expect(status).to eq 422
    end

    example "Create Zip Code do not provide zip_code params", document: false do
      do_request

      expect(status).to eq 400
      expect(response_body).to eq '{"message":"Invalid Parameter: zip_code","errors":{"zip_code":"Parameter is required"}}'
    end
  end

  get "/api/v1/zip_codes/:zip.json" do
    parameter :zip, "Zip", scope: :zip_code, required: true

    let(:zip_code) { create(:zip_code) }

    example "Read Zip Code" do
      do_request(zip: zip_code.zip)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
        zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
    end

    example "Read Zip Code that does not exist", document: false do
      do_request(zip: '12345-6789')

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Read Zip Code provide invalid format zip", document: false do
      do_request(zip: '1234')
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: zip'
      expect(json_response[:errors][:zip]).to eq 'Parameter must match format (?-mix:\A\d{5}(?:-\d{4})?\Z)'
    end
  end

  put "/api/v1/zip_codes/:id.json" do
    header "Content-Type", "application/json"

    parameter :id, "Record ID", required: true
    parameter :street_name, "Street name", scope: :zip_code
    parameter :building_number, "Building number", scope: :zip_code
    parameter :city, "City", scope: :zip_code
    parameter :state, "State", scope: :zip_code
    let(:raw_post) { params.to_json }

    let(:zip_code) { create(:zip_code) }
    let(:valid_attributes) { attributes_for(:zip_code) }

    example "Update Zip Code" do
      do_request(id: zip_code.id, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 200
      expect(json_response[:zip_code].values_at(:zip, :street_name, :building_number, :city, :state)).to eq(
        valid_attributes.values_at(:zip, :street_name, :building_number, :city, :state))
      expect(zip_code.reload.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
    end

    example "Update Zip Code that does not exist", document: false do
      do_request(id: 800, zip_code: valid_attributes)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Update Zip Code provide to big ID number", document: false do
      do_request(id: 3000000000, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end

    example "Update Zip Code provide not Hash zip_code params", document: false do
      do_request(id: zip_code.id, zip_code: "STRING")

      expect(status).to eq 200
    end

    example "Update Zip Code do not provide zip_code params", document: false do
      do_request(id: zip_code.id)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: zip_code'
      expect(json_response[:errors][:zip_code]).to eq 'Parameter is required'
    end
  end

  delete "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    example "Delete Zip Code" do
      do_request(id: zip_code.id)

      expect(status).to eq 200
      expect(ZipCode.where(id: zip_code.id)).to be_empty
    end

    example "Delete Zip Code that does not exist", document: false do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Delete Zip Code provide to big ID number", document: false do
      do_request(id: 3000000000)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end
  end
end
```

## <a name="summary"></a>Рэзюмэ

Мы прадугледзелі апрацоўку памылак і адпаведныя HTTP адказы з кодамі стану і паведамленнямі пра памылкі ў фармаце JSON (мы выкарыстоўвалі для апрацоўкі памылак здольнасць `sinatra`). Мы выкарыстоўвалі гем [sinatra-param](https://github.com/mattt/sinatra-param) для праверкі параметраў і пераўтварэнні тыпаў у `sinatra`.
