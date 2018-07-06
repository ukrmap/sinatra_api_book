Глава #7. Аутентификация и авторизация
======================================

Мы должны думать о безопасности и правах доступа, в настоящее время веб-сервис Zip-кодов открыт для общественности - любой может создать новый Zip-код и обновить или удалить вовсе. Давайте создадим авторизацию пользователей используя веб-сервис `users`, созданный в главе №4. Все зарегистрированные пользователи смогут создавать новые ZIP-коды и только пользователь-администратор сможет обновить или удалить существующий Zip-код.

Клиент веб-сервиса Zip-кодов должен передать токен в HTTP заголовке для аутентификации. Веб-сервис Zip-кодов будет использовать этот же токен, чтобы получить идентификатор пользователя из веб-сервиса `users`. Так внутренне, веб-сервис Zip-кодов выполняет (будет выполнять) еще один HTTP запрос - это вызывает незначительную потерю производительности. Вы должны принимать во внимание затраты на выполнение внутренних HTTP запросов при разработке архитектуры сервиса.

## <a name="remote-calls-to-users-service"></a>Удаленные вызовы к `users` сервису

Помните `users` сервис, который был создан в главе №4 - мы планируем использовать его для аутентификации в сервисе Zip-кодов. Мы сделаем некоторое исследование о том, как мы будем использовать `users` сервис из `ruby` кода.

Нам нужен HTTP-клиент гем, например [faraday](https://github.com/lostisland/faraday). Установите его пожалуйста.

    $ gem install faraday

Затем запустите пожалуйста веб-сервис `users` на порту 4545 (выполните в терминале из папки `users`)

    $ ruby service.rb -p 4545

Откройте новую вкладку терминала и запустите `irb` сессию (напечатайте `irb` и нажмите `Enter`). Вот наша сессия:

    $ irb
    > require "faraday"
    => true
    > response = Faraday.new("http://localhost:4545/api/v1/users/me.json",
    > headers: { "Authorization" => "OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86" }).get
    => #<Faraday::Response:0x0000010199d1d8 @on_complete_callbacks=[], @env=#<Faraday::Env @method=:get @body="{\"user\":{\"type\":\"RegularUser\"}}" @url=#<URI::HTTP:0x000001019abc60 URL:http://localhost:4545/api/v1/users/me.json> @request=#<Faraday::RequestOptions (empty)> @request_headers={"Authorization"=>"OAuth b259ca1339e168b8295287648271acc94a9b3991c608a3217fecc25f369aaa86", "User-Agent"=>"Faraday v0.9.1"} @ssl=#<Faraday::SSLOptions (empty)> @response_headers={"content-type"=>"application/json", "content-length"=>"31", "x-content-type-options"=>"nosniff", "connection"=>"close", "server"=>"thin"} @status=200>>
    > response.status
    => 200
    > response.body
    => "{\"user\":{\"type\":\"RegularUser\"}}"

Мы использовали корректный токен для `"RegularUser"`. Давайте попробуем некорректный токен.

    $ irb
    > require "faraday"
    => true
    > response = Faraday.new("http://localhost:4545/api/v1/users/me.json",
    > headers: { "Authorization" => "OAuth incorrect" }).get
    => #<Faraday::Response:0x00000101097a70 @on_complete_callbacks=[], @env=#<Faraday::Env @method=:get @body="{\"message\":\"Invalid or expired token\"}" @url=#<URI::HTTP:0x0000010183a020 URL:http://localhost:4545/api/v1/users/me.json> @request=#<Faraday::RequestOptions (empty)> @request_headers={"Authorization"=>"OAuth incorrect", "User-Agent"=>"Faraday v0.9.1"} @ssl=#<Faraday::SSLOptions (empty)> @response_headers={"content-type"=>"application/json", "content-length"=>"38", "x-content-type-options"=>"nosniff", "connection"=>"close", "server"=>"thin"} @status=401>>
    > response.status
    => 401
    > response.body
    => "{\"message\":\"Invalid or expired token\"}"

Мы можем реализовать ту же логику в сервисе Zip-кодов. HTTP `"Authorization"` заголовок доступен в `sinatra` как `request.env['HTTP_AUTHORIZATION']`.

## <a name="remote-authentication"></a>Удаленная аутентификация

Мы будем использовать гем авторизации для Ruby on Rails - [cancan](https://github.com/ryanb/cancan) и `sinatra` обёртку для него - [sinatra-can](https://github.com/shf/sinatra-can). Нам нужен будет гем [fakeweb](https://github.com/facewatch/fakeweb) для эмуляции запросов к `users` сервису в тестах. Пожалуйста, добавьте три этих гема в `Gemfile`:

```ruby
source 'https://rubygems.org'

gem 'rake'
gem 'sinatra', require: 'sinatra/main'
gem 'rack-contrib', git: 'https://github.com/rack/rack-contrib'
gem 'pg'
gem 'activerecord'
gem 'protected_attributes'
gem 'sinatra-activerecord'
gem 'sinatra-param'

# Simple, but flexible HTTP client library, with support for multiple backends.
gem 'faraday'

# CanCan wrapper for Sinatra. (CanCan is Authorization Gem for Ruby on Rails)
gem 'sinatra-can'

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

  # A test helper for faking responses to web requests
  gem 'fakeweb'
end
```

И выполните (из папки `zip_codes` в терминале)

    $ bundle install

Мы будем использовать вспомогательные методы из `sinatra-can`, чтобы настроить `current_user` и `current_ ability`. Мы также должны добавить обработчики ошибок для 401 и 403. Пожалуйста, обновите файл `application.rb`.

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

# Set up current_user
user do
  if request.env['HTTP_AUTHORIZATION'].present?
    response = Faraday.new("http://localhost:4545/api/v1/users/me.json",
      headers: { 'Authorization' => request.env['HTTP_AUTHORIZATION'] }).get
    halt 401 if response.status == 401 # go to error 401 handler
    OpenStruct.new(JSON.parse(response.body)['user']) if response.success?
  end
end

# Set up current_ability
ability do |user|
  can :manage, ZipCode if user.present? && user.type == 'AdminUser'
  can :create, ZipCode if user.present? && user.type == 'RegularUser'
end

# Client uses incorrect token
error(401) { '{"message":"Invalid or expired token"}' }

# Operation forbidden (user can be logged in with token or public)
error(403) { '{"message":"Access Forbidden"}' }

error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }
error(ActiveRecord::RecordInvalid) do
  [422, { message: "Validation errors occurred",
          errors:  env['sinatra.error'].record.errors.messages }.to_json ]
end
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

![](../static/images/cancan.jpg)

И проверка `ability` в файле `app/controllers/zip_codes_controller.rb` вспомогательным методод `authorize!` из `sinatra-can`.

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  authorize! :create, zip_code # Authorize create new ZipCode
  zip_code.save!
  status 201
  zip_code.to_json
end

# No authorization - available for not logged in users
get "/api/v1/zip_codes/:zip.json" do
  param :zip, String, format: /\A\d{5}(?:-\d{4})?\Z/
  zip_code = ZipCode.find_by_zip!(params[:zip])
  zip_code.to_json
end

put "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  param :zip_code, Hash, required: true
  zip_code = ZipCode.find(params[:id])
  authorize! :update, zip_code # Authorize update ZipCode
  zip_code.update_attributes!(params[:zip_code]) if params[:zip_code].any?
  zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  zip_code = ZipCode.find(params[:id])
  authorize! :destroy, zip_code # Authorize destroy ZipCode
  zip_code.destroy!
end
```

## <a name="emulate-remote-authentication-in-tests"></a>Эмуляция удаленной аутентификации в тестах

Мы должны исправить `acceptance` тесты. Есть несколько тестов, которые не проходят - все из-за попытки публичного доступа к защищенным действиям - 403 ошибка при попытке изменить Zip-код.

Во-первых, вы должны быть уведомлены, если приложение выполняет внешние запросы HTTP в тестах (в нашем случае к `users` сервису). Измените файл `spec/spec_helper.rb` - добавьте строку `FakeWeb.allow_net_connect = false`

```ruby
ENV['RACK_ENV'] = 'test'
require File.expand_path("../../application", __FILE__)

FactoryGirl.find_definitions

# Catch when requests are made for unregistered URIs.
FakeWeb.allow_net_connect = false

RSpec.configure do |config|
  config.include Rack::Test::Methods
  config.include FactoryGirl::Syntax::Methods
  config.default_formatter = 'doc' if config.files_to_run.one?

  def app
    Sinatra::Application
  end

  config.before(:suite) do
    DatabaseCleaner.clean_with :truncation
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
end

require "rspec_api_documentation/dsl"

RspecApiDocumentation.configure do |config|
  config.docs_dir = Pathname.new(Sinatra::Application.root).join("doc")
  config.app = Sinatra::Application
  config.api_name = "Zip-Codes API"
  config.format = :html
  config.curl_host = 'https://zipcodes.example.com'
  config.curl_headers_to_filter = %w(Host Cookie)
end
```

В тестах мы должны эмулировать ситуацию, когда клиент использует токен. Когда клиент не передает OAuth токен - приложение не должно выполнять запрос к `users` сервису.

```ruby
header "Authorization", 'OAuth abcdefgh12345678'
```

Метод [header](https://github.com/zipmark/rspec_api_documentation#header) определен в геме `rspec_api_documentation`.

И имитируем HTTP ответ из `users` сервиса

```ruby
FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
  body: '{"user":{"id":1,"type":"AdminUser"}}')
```

Вот обновленная версия файла `spec/acceptance/zip_codes_spec.rb`

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

    context "Public User", document: false do
      example "Create Zip Code" do
        do_request(zip_code: valid_attributes)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 403
        expect(json_response).to eq(message: "Access Forbidden")
        expect(new_zip_code).to be_nil
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")' do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Create Zip Code by Regular User" do
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

    context 'Admin User', document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"AdminUser"}}') }

      example "Create Zip Code" do
        do_request(zip_code: valid_attributes)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 201
        expect(json_response[:zip_code].values_at(*valid_attributes.keys)).to eq valid_attributes.values
        expect(new_zip_code).to be_present
        expect(new_zip_code.attributes.values_at(*valid_attributes.keys.map(&:to_s))).to eq valid_attributes.values
      end
    end
  end

  get "/api/v1/zip_codes/:zip.json" do
    parameter :zip, "Zip", scope: :zip_code, required: true

    let(:zip_code) { create(:zip_code) }

    context "Public User" do
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

    context 'Regular User (authenticated user with type "RegularUser")', document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Read Zip Code" do
        do_request(zip: zip_code.zip)
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
        do_request(zip: zip_code.zip)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 200
        expect(json_response[:zip_code].values_at(:id, :zip, :street_name, :building_number, :city, :state)).to eq(
          zip_code.attributes.values_at('id', 'zip', 'street_name', 'building_number', 'city', 'state'))
      end
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

    context "Public User", document: false do
      example "Update Zip Code" do
        do_request(id: zip_code.id, zip_code: valid_attributes)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 403
        expect(json_response).to eq(message: "Access Forbidden")
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")', document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Update Zip Code" do
        do_request(id: zip_code.id, zip_code: valid_attributes)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 403
        expect(json_response).to eq(message: "Access Forbidden")
      end
    end

    context "Admin User" do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"AdminUser"}}') }

      example "Update Zip Code by Admin" do
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
  end

  delete "/api/v1/zip_codes/:id.json" do
    parameter :id, "Record ID", required: true

    let(:zip_code) { create(:zip_code) }

    context "Public User", document: false do
      example "Delete Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 403
        expect(json_response).to eq(message: "Access Forbidden")
        expect(ZipCode.where(id: zip_code.id)).to be_present
      end
    end

    context 'Regular User (authenticated user with type "RegularUser")', document: false do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"RegularUser"}}') }

      example "Delete Zip Code" do
        do_request(id: zip_code.id)
        json_response = JSON.parse(response_body, symbolize_names: true)

        expect(status).to eq 403
        expect(json_response).to eq(message: "Access Forbidden")
        expect(ZipCode.where(id: zip_code.id)).to be_present
      end
    end

    context "Admin User" do
      header "Authorization", 'OAuth abcdefgh12345678'
      before { FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
        body: '{"user":{"id":1,"type":"AdminUser"}}') }

      example "Delete Zip Code by Admin" do
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
end
```

Мы протестировали все случаи, когда пользователь Admin, обычный пользователь или публичный пользователь использует API сервиса Zip-кодов. Теперь мы можем сделать рефакторинг с уверенностью, что сервис будет работать должным образом после изменений (если после изменений пройдут тесты).

## <a name="refactoring"></a>Рефакторинг

Мы можем перенести `ability` в отдельный класс - `app/models/ability.rb`, `cancan` (`sinatra-can`) использует этот класс по умолчанию.

```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    return unless user

    can :manage, ZipCode if user.type == 'AdminUser'
    can :create, ZipCode if user.type == 'RegularUser'
  end
end
```

И удалите блок `ability` из файла` application.rb`

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

# Set up current_user
user do
  if request.env['HTTP_AUTHORIZATION'].present?
    response = Faraday.new("http://localhost:4545/api/v1/users/me.json",
      headers: { 'Authorization' => request.env['HTTP_AUTHORIZATION'] }).get
    halt 401 if response.status == 401 # go to error 401 handler
    OpenStruct.new(JSON.parse(response.body)['user']) if response.success?
  end
end

# Client uses incorrect token
error(401) { '{"message":"Invalid or expired token"}' }

# Operation forbidden (user can be logged in with token or public)
error(403) { '{"message":"Access Forbidden"}' }

error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }
error(ActiveRecord::RecordInvalid) do
  [422, { message: "Validation errors occurred",
          errors:  env['sinatra.error'].record.errors.messages }.to_json ]
end
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

Также мы можем использовать вспомогательный метод `load_and_authorize!` из `sinatra-can`. Это позволяет найти запись по `params[:id]` и проверить доступность в одной строке.

Таким образом, мы можем написать так:

```ruby
load_and_authorize! ZipCode
```

Вместо этого:

```ruby
@zip_code = ZipCode.find(params[:id])
authorize! :update, @zip_code
```

Или это:

```ruby
@zip_code = ZipCode.find(params[:id])
authorize! :destroy, @zip_code
```

Но как `sinatra-can` догадается, для каких `ability` проверять доступ (`update` или `destroy`)? `load_and_authorize` проверяет `ability` в соответствии с методом запроса: `PUT` для `:update`, `DELETE` для `:destroy`. Также обратите внимание, что `load_and_authorize!` инициализирует переменную экземпляра (имя с префиксом `@`).

Обновленная версия `app/controllers/zip_codes_controller.rb`

```ruby
post "/api/v1/zip_codes.json" do
  param :zip_code, Hash, required: true
  zip_code = ZipCode.new(params[:zip_code])
  authorize! :create, zip_code
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
  param :id, Integer, max: 2147483647
  param :zip_code, Hash, required: true
  load_and_authorize! ZipCode # load @zip_code and authorize! :update, @zip_code
  @zip_code.update_attributes!(params[:zip_code]) if params[:zip_code].any?
  @zip_code.to_json
end

delete "/api/v1/zip_codes/:id.json" do
  param :id, Integer, max: 2147483647
  load_and_authorize! ZipCode # load @zip_code and authorize! :destroy, @zip_code
  @zip_code.destroy!
end
```

Если запись не найдена `sinatra-can` вызывает `error(404)` (почти тоже, что `halt(404)`). Мы должны добавить JSON-сообщение об ошибке 404 так же как при перехвате `ActiveRecord::RecordNotFound`.

В файле `application.rb` замените строку

```ruby
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }`
```

на

```ruby
error(404, ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }
```

Вот полный `application.rb`

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

user do
  if request.env['HTTP_AUTHORIZATION'].present?
    response = Faraday.new("http://localhost:4545/api/v1/users/me.json",
      headers: { 'Authorization' => request.env['HTTP_AUTHORIZATION'] }).get
    halt 401 if response.status == 401 # go to error 401 handler
    OpenStruct.new(JSON.parse(response.body)['user']) if response.success?
  end
end

error(401) { '{"message":"Invalid or expired token"}' }
error(403) { '{"message":"Access Forbidden"}' }
error(404, ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }
error(ActiveRecord::RecordInvalid) do
  [422, { message: "Validation errors occurred",
          errors:  env['sinatra.error'].record.errors.messages }.to_json ]
end
error { '{"message":"An internal server error occurred. Please try again later."}' }
```

Не забудьте провести тесты.

## <a name="summary"></a>Резюме

Мы создали удаленную аутентификацию с помощью веб-сервиса `users` используя HTTP-клиент библиотеку [faraday](https://github.com/lostisland/faraday). Мы использовали [fakeweb](https://github.com/facewatch/fakeweb) для тестирования удаленной аутентификации (эмуляция ответа от веб-сервиса `users`).

Мы использовали [sinatra-can](https://github.com/shf/sinatra-can) (`sinatra` обертка для [cancan](https://github.com/ryanb/cancan)) для авторизации. Мы расширили сервис Zip-кодов с 401 и 403 кодами состояния HTTP (и сообщениями об ошибках).
