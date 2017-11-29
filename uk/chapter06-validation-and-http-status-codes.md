Глава #6. Валідація та коди стану HTTP
======================================

Обработка некорректного ввода данных является важным аспектом веб-сервиса, а также сообщения об ошибках для клиента. [Коди стану HTTP](https://uk.wikipedia.org/wiki/%D0%A1%D0%BF%D0%B8%D1%81%D0%BE%D0%BA_%D0%BA%D0%BE%D0%B4%D1%96%D0%B2_%D1%81%D1%82%D0%B0%D0%BD%D1%83_HTTP) - це загальна проста концепція для повідомлення клієнта про статус обробки HTTP запиту. Повідомлення про помилку в форматі JSON доповнює HTTP-відповідь і уточнює помилку для кінцевих користувачів.

Ось список кодів стану HTTP, які ми будемо використовувати:

* 200, 201 - Успішні операції: Добре - OK і Створено (201 - для `POST` запитів)
* 404      - Не знайдено
* 400, 422 - Неправильний запит, Необроблюваний екземпляр
* 401, 403 - Несанкціонований доступ, Заборонено
* 500      - Внутрішня помилка сервера

400 і 422 дуже схожі, ми будемо використовувати 400, щоб повідомити про невірні параметри запиту в цілому (невірний тип або значення поза діапазоном) і 422 для помилок валідації.

401 і 403 також схожі. Ми будемо використовувати 401, якщо користувач відправляє невірний або з вичерпаним терміном дії токен для аутентифікації. І 403, щоб повідомити користувача (який можливо увійшов в систему) про помилку авторизації (заборона на виконання операції).

## <a name="500-internal-server-error"></a>500 - Внутрішня помилка сервера

При поточній реалізації веб-сервісу, якщо відбувається непередбачена помилка, сервіс повертає 500-у помилку без тіла відповіді. Розробляючи JSON-сервіс, ми повинні передбачити повідомлення для клієнта з відповідним повідомленням про помилку в форматі JSON (у файлі `application.rb`).

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

## <a name="404-not-found"></a>404 - Не знайдено

Якщо клієнт надає ідентифікатор (ID) Zip-коду, який не існує в базі даних, ActiveRecord метод `find` повертає помилку `ActiveRecord::RecordNotFound` і веб-сервіс повертає 500-й код стану HTTP. Це не добре. Сервіс повинен повертати 404-й код статусу і відповідне повідомлення про помилку.

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

## <a name="422-unprocessable-entity"></a>422 - Необроблюваний екземпляр

Якщо ми спробуємо зберегти модель з невірними даними (наприклад, неправильний формат ZIP) ActiveRecord поверне помилку `ActiveRecord::RecordInvalid`. У цьому випадку сервіс повинен повертати 422-ий код статусу HTTP і повідомлення про специфічні помилки валідації.

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

## <a name="tests"></a>Тести

Ми можемо перевірити обробку помилок в `acceptance` тестах. Файл `spec/acceptance/zip_codes_spec.rb`

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

## <a name="400-bad-request"></a>400 - Неправильний запит

Іноді нам потрібно забезпечити правильність типів параметрів, які передає клієнт. Ми можемо запобігти взаємодії з базою даних при некоректних вхідних даних і негайно повернути повідомлення про помилку клієнту. Ми будемо використовувати гем [sinatra-param](https://github.com/mattt/sinatra-param) для перевірки та перетворення типів параметрів у `sinatra`.

Будь ласка, додайте рядок `gem 'sinatra-param'` в` Gemfile`

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

І виконайте `bundle install`

Якщо клієнт надає параметр `zip_code` неправильного типу (не `Hash`) веб-сервіс повертає помилку 500 (наприклад, ``NoMethodError: undefined method `stringify_keys' for "STRING":String`` для типу `String`). Ми можемо запобігти цій поведінці за допомогою наступного декларативного коду у файлі `app/controllers/zip_codes_controller.rb`.

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

Ми можемо піти далі і перевіряти регулярним виразом параметр `zip` в` get` роуті. Це запобігає запит до бази даних, якщо параметр має невірний формат. І веб-сервіс повертає код HTTP статусу 400 замість 404, а також більш відповідне повідомлення про помилку (`"Параметр повинен відповідати формату"` замість `"Запис не знайдено"`, яке передбачає, що запис можливо було видалено).

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

У мене є обмеження для цілого типу в базі даних. Якщо клієнт надає ідентифікатор більший за `2147483647` (що в двійковій системі дорівнює `111111111111111111111111111111`) `activerecord` повертає помилку `PostgreSQL` та веб-сервіс відповідає з 500-ю HTTP помилкою. Можна уникнути цього з наступною фільтрацією параметрів.

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

Оновлені тести (файл `spec/acceptance/zip_codes_spec.rb`)

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

Ми передбачили обробку помилок і відповідні HTTP відповіді з кодами стану і повідомленнями про помилки в форматі JSON (ми використали для обробки помилок здатність `sinatra`). Ми використали гем [sinatra-param](https://github.com/mattt/sinatra-param) для перевірки параметрів і перетворення типів в `sinatra`.
