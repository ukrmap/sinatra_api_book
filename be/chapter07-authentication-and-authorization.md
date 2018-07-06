Кіраўнік #7. Аўтэнтыфікацыя і аўтарызацыя
=========================================

Мы павінны думаць пра бяспеку і правах доступу, у цяперашні час вэб-сэрвіс Zip-кодаў адкрыты для грамадскасці - любы можа стварыць новы Zip-код і абнавіць або выдаліць зусім. Давайце створым аўтарызацыю карыстальнікаў выкарыстоўваючы вэб-сэрвіс `users`, створаны ў главе №4. Усе зарэгістраваныя карыстальнікі змогуць ствараць новыя ZIP-коды і толькі карыстач-адміністратар зможа абнавіць або выдаліць існуючы Zip-код.

Кліент вэб-сэрвісу Zip-кодаў павінен перадаць токен ў HTTP загалоўку для аўтэнтыфікацыі. Вэб-сэрвіс Zip-кодаў будзе выкарыстоўваць гэты ж токен, каб атрымаць ідэнтыфікатар карыстальніка з вэб-сэрвісу `users`. Так ўнутрана, вэб-сэрвіс Zip-кодаў выконвае (будзе выконваць) яшчэ адзін HTTP запыт - гэта выклікае нязначную страту прадукцыйнасці. Вы павінны прымаць да ўвагі затраты на выкананне ўнутраных HTTP запытаў пры распрацоўцы архітэктуры сэрвісу.

## <a name="remote-calls-to-users-service"></a>Выдаленыя выклікі да `users` сэрвісу

Памятаеце `users` сэрвіс, які быў створаны ў главе №4 - мы плануем выкарыстоўваць яго для ідэнтыфікацыі ў сэрвісе Zip-кодаў. Мы зробім некаторы даследаванне аб тым, як мы будзем выкарыстоўваць `users` сэрвіс з `ruby` кода.

Нам патрэбны HTTP-кліент гем, напрыклад [faraday](https://github.com/lostisland/faraday). Усталюйце яго калі ласка.

    $ gem install faraday

Затым запусціце ласка вэб-сэрвіс `users` на порце 4545 (выканайце ў тэрмінале з папкі `users`)

    $ ruby service.rb -p 4545

Адкрыйце новую ўкладку тэрмінала і запусціце `irb` сесію (надрукуйце `irb` і націсніце `Enter`). Вось наша сесія:

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

Мы выкарыстоўвалі карэктны токен для `"RegularUser"`. Давайце паспрабуем некарэктны токен.

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

Мы можам рэалізаваць тую ж логіку ў сэрвісе Zip-кодаў. HTTP `"Authorization"` загаловак даступны ў `sinatra` як `request.env['HTTP_AUTHORIZATION']`.

## <a name="remote-authentication"></a>Аддаленая аўтэнтыфікацыя

Мы будзем выкарыстоўваць гем аўтарызацыі для Ruby on Rails - [cancan](https://github.com/ryanb/cancan) і `sinatra` абгортку для яго - [sinatra-can](https://github.com/shf/ sinatra-can). Нам патрэбны будзе гем [fakeweb](https://github.com/facewatch/fakeweb) для эмуляцыі запытаў да `users` сэрвісу ў тэстах. Калі ласка, дадайце тры гэтых гема ў `Gemfile`:

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

І выканайце (з папкі `zip_codes` ў тэрмінале)

    $ bundle install

Мы будзем выкарыстоўваць дапаможныя метады з `sinatra-can`, каб наладзіць `current_user` і `current_ability`. Мы таксама павінны дадаць апрацоўшчыкі памылак для 401 і 403. Калі ласка, абновіце файл `application.rb`.

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

І праверка `ability` ў файле `app/controllers/zip_codes_controller.rb` дапаможным методод `authorize!` з `sinatra-can`.

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

## <a name="emulate-remote-authentication-in-tests"></a>Эмуляцыя выдаленай ідэнтыфікацыі ў тэстах

Мы павінны выправіць `acceptance` тэсты. Ёсць некалькі тэстаў, якія не праходзяць - усё з-за спробы публічнага доступу да абароненых дзеянням - 403 памылка пры спробе змяніць Zip-код.

Па-першае, вы павінны быць апавешчаныя, калі дадатак выконвае знешнія запыты HTTP ў тэстах (у нашым выпадку да `users` сэрвісу). Зменіце файл `spec/spec_helper.rb` - дадайце радок `FakeWeb.allow_net_connect = false`

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

У тэстах мы павінны эмуляваць сітуацыю, калі кліент выкарыстоўвае токен. Калі кліент не перадае OAuth токен - прыкладанне не павінна выконваць запыт да `users` сэрвісу.

```ruby
header 'Authorization', 'OAuth abcdefgh12345678'
```

Метад [header](https://github.com/zipmark/rspec_api_documentation#header) вызначаны ў геме `rspec_api_documentation`.

І імітуючы HTTP адказ з `users` сэрвісу

```ruby
FakeWeb.register_uri(:get, "http://localhost:4545/api/v1/users/me.json",
  body: '{"user":{"id":1,"type":"AdminUser"}}')
```

Вось абноўленая версія файла `spec/acceptance/zip_codes_spec.rb`

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

Мы пратэставалі ўсе выпадкі, калі карыстальнік Admin, звычайны карыстальнік ці публічны карыстальнік выкарыстоўвае API сэрвісу Zip-кодаў. Цяпер мы можам зрабіць рэфактарынг з упэўненасцю, што сэрвіс будзе працаваць належным чынам пасля зменаў (калі пасля зменаў пройдуць тэсты).

## <a name="refactoring"></a>Рэфактарынг

Мы можам перанесці `ability` ў асобны клас - `app/models/ability.rb`, `cancan` (`sinatra-can`) выкарыстоўвае гэты клас па змаўчанні.

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

І выдаліце блок `ability` з файла `application.rb`

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

Таксама мы можам выкарыстоўваць дапаможны метад `load_and_authorize!` з `sinatra-can`. Гэта дазваляе знайсці запіс па `params[:id]` і праверыць даступнасць ў адным радку.

Такім чынам, мы можам напісаць так:

```ruby
load_and_authorize! ZipCode
```

Замест гэтага:

```ruby
@zip_code = ZipCode.find(params[:id])
authorize! :update, @zip_code
```

Ці гэта:

```ruby
@zip_code = ZipCode.find(params[:id])
authorize! :destroy, @zip_code
```

ле як `sinatra-can` здагадаецца, для якіх `ability` правяраць доступ (`update` або `destroy`)? `load_and_authorize` правярае `ability` ў адпаведнасці з метадам запыту: `PUT` для `:update`, `DELETE` для `:destroy`. Таксама зьвярніце ўвагу, што `load_and_authorize!` ініцыялізуе зменную асобніка (імя з прэфіксам `@`).

Абноўленая версія `app/controllers/zip_codes_controller.rb`

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

Калі запіс не знойдзена `sinatra-can` выклікае `error(404)` (амаль таксама, што `halt(404)`). Мы павінны дадаць JSON-паведамленне пра памылку 404 гэтак жа як пры перахопе `ActiveRecord::RecordNotFound`.

У файле `application.rb` заменіце радок

```ruby
error(ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }`
```

на

```ruby
error(404, ActiveRecord::RecordNotFound) { [404, '{"message":"Record not found"}'] }
```

KUось поўны `application.rb`

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

Не забудзьцеся правесці тэсты.

## <a name="summary"></a>Рэзюмэ

Мы стварылі выдаленую аўтэнтыфікацыю з дапамогай вэб-сэрвісу `users` выкарыстоўваючы HTTP-кліент бібліятэку [faraday](https://github.com/lostisland/faraday). Мы выкарыстоўвалі [fakeweb](https://github.com/facewatch/fakeweb) для тэставання выдаленай аўтэнтыфікацыі (эмуляцыя адказу ад вэб-сэрвісу `users`).

Мы выкарыстоўвалі [sinatra-can](https://github.com/shf/sinatra-can) (`sinatra` абгортка для [cancan](https://github.com/ryanb/cancan)) для аўтарызацыі. Мы пашырылі сэрвіс Zip-кодаў з 401 і 403 кодамі стану HTTP (і паведамленнямі пра памылкі).
