Глава #6. Валидация и коды состояния HTTP
=========================================

Обробка некоректного введення даних є важливим аспектом функционирования веб-сервісу, а також повідомлення про помилки для клієнта. [Коды состояния HTTP](https://ru.wikipedia.org/wiki/%D0%A1%D0%BF%D0%B8%D1%81%D0%BE%D0%BA_%D0%BA%D0%BE%D0%B4%D0%BE%D0%B2_%D1%81%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D1%8F_HTTP) - это общая простая концепция для уведомления клиента о статусе обработки HTTP запроса. Сообщение об ошибке в формате JSON дополняет HTTP-ответ и уточняет ошибку для конечных пользователей.

Вот список кодов состояния HTTP, которые мы будем использовать:

* 200, 201 - успешно: «хорошо» - OK и «создано» (201 - для `POST` запросов)
* 404      - «не найдено»
* 400, 422 - «плохой, негодный запрос», «необрабатываемый экземпляр»
* 401, 403 - «неавторизован», «запрещено»
* 500      - «внутренняя ошибка сервера»

400 и 422 очень похожи, мы будем использовать 400, чтобы уведомить о неверных параметрах запроса в целом (неверный тип или значение вне диапазона) и 422 для ошибок валидации.

401 и 403 также схожи. Мы будем использовать 401, если пользователь отправляет неверный или с истекшим сроком действия токен для аутентификации. И 403, чтобы уведомить пользователя (который возможно вошел в систему) об ошибке авторизации (запрет на выполнение операции).

## <a name="500-internal-server-error"></a>500 - Внутренняя ошибка сервера

При текущей реализации веб-сервиса, если происходит непредвиденная ошибка, сервис возвращает 500-ю ошибку без тела ответа. Разрабатывая JSON-сервис, мы должны предусмотреть уведомление для клиента с соответствующим сообщением об ошибке в формате JSON (в файле `application.rb`).

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

## <a name="404-not-found"></a>404 - Не найдено

Если клиент предоставляет идентификатор (ID) Zip-кода, который не существует в базе данных, ActiveRecord метод `find` возвращает ошибку `ActiveRecord::RecordNotFound` и веб-сервис возвращает 500-й код состояния HTTP. Это не хорошо. Сервис должен возвращать 404-й код статуса и соответствующее сообщение об ошибке.

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

## <a name="422-unprocessable-entity"></a>422 - Необрабатываемый экземпляр

Если мы попытаемся сохранить модель с неверными данными (например, неправильный формат ZIP) ActiveRecord вернёт ошибку `ActiveRecord::RecordInvalid`. В этом случае сервис должен возвращать 422-ой код статуса HTTP и сообщение о специфических ошибках валидации.

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

## <a name="tests"></a>Тесты

Мы можем проверить обработку ошибок в `acceptance` тестах. Файл `spec/acceptance/zip_codes_spec.rb`

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

    example "Create Zip Code with invalid params", document: nil do
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

    example "Read Zip Code that does not exist", document: nil do
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

    example "Update Zip Code that does not exist", document: nil do
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

    example "Delete Zip Code that does not exist", document: nil do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end
  end
end
```

## <a name="400-bad-request"></a>400 - Неправильный запрос

Иногда нам нужно обеспечить правильность типов параметров, которые передаёт клиент. Мы можем предотвратить взаимодействие с базой данных при некорректных входных данных и немедленно вернуть сообщение об ошибке клиенту. Мы будем использовать гем [sinatra-param](https://github.com/mattt/sinatra-param) для проверки и преобразования типов параметров в `sinatra`.

Пожалуйста, добавьте строку `gem 'sinatra-param'` в `Gemfile`

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

И выполните `bundle install`

Если клиент предоставляет параметр `zip_code` неправильного типа (не `Hash`) веб-сервис возвращает ошибку 500 (например, ``NoMethodError: undefined method `stringify_keys' for "STRING":String`` для типа `String`). Мы можем предотвратить это поведение с помощью следующего декларативного кода в файле `app/controllers/zip_codes_controller.rb`.

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

Мы можем пойти дальше и проверять регулярным выражением параметр `zip` в `get` роуте. Это предотвращает запрос к базе данных, если параметр имеет неверный формат. И веб-сервис возвращает код HTTP статуса 400 вместо 404, а также более подходящее сообщение об ошибке (`"Параметр должен соответствовать формату"` вместо `"Запись не найдена"`, которое предусматривает, что запись возможно была удалена).

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

У меня есть ограничения для целого типа в базе данных. Если клиент предоставляет идентификатор больше чем `2147483647` (что в двоичной системе равно `111111111111111111111111111111`) `activerecord` возвращает ошибку ` PostgreSQL` и веб-сервис отвечает с 500-й HTTP ошибкой. Можно избежать этого с последующей фильтрацией параметров.

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

Обновленные тесты (файл `spec/acceptance/zip_codes_spec.rb`)

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

    example "Create Zip Code with invalid params", document: nil do
      do_request(zip_code: { zip: "1234" })

      expect(status).to eq 422
      expect(response_body).to eq '{"message":"Validation errors occurred","errors":{"zip":["is invalid"]}}'
      expect(new_zip_code).to be_nil
    end

    example "Create Zip Code provide not Hash zip_code params", document: nil do
      do_request(zip_code: "STRING")

      expect(status).to eq 422
    end

    example "Create Zip Code do not provide zip_code params", document: nil do
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

    example "Read Zip Code that does not exist", document: nil do
      do_request(zip: '12345-6789')

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Read Zip Code provide invalid format zip", document: nil do
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

    example "Update Zip Code that does not exist", document: nil do
      do_request(id: 800, zip_code: valid_attributes)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Update Zip Code provide to big ID number", document: nil do
      do_request(id: 3000000000, zip_code: valid_attributes)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end

    example "Update Zip Code provide not Hash zip_code params", document: nil do
      do_request(id: zip_code.id, zip_code: "STRING")

      expect(status).to eq 200
    end

    example "Update Zip Code do not provide zip_code params", document: nil do
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

    example "Delete Zip Code that does not exist", document: nil do
      do_request(id: 800)

      expect(status).to eq 404
      expect(response_body).to eq '{"message":"Record not found"}'
    end

    example "Delete Zip Code provide to big ID number", document: nil do
      do_request(id: 3000000000)
      json_response = JSON.parse(response_body, symbolize_names: true)

      expect(status).to eq 400
      expect(json_response[:message]).to eq 'Invalid Parameter: id'
      expect(json_response[:errors][:id]).to eq 'Parameter cannot be greater than 2147483647'
    end
  end
end
```

## <a name="summary"></a>Резюме

Мы предусмотрели обработку ошибок и соответствующие HTTP ответы с кодами состояния и сообщениями об ошибках в формате JSON (мы использовали для обработки ошибок способность `sinatra`). Мы использовали гем [sinatra-param](https://github.com/mattt/sinatra-param) для проверки параметров и преобразования типов в `sinatra`.
